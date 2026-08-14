# Real-Model Stability Nudge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add one budget-aware retrieval nudge after the first tool call so the real model re-queries before abstaining, while preserving mock score, safety, provenance, trace, and tool budget.

**Architecture:** Extend only `BudgetPolicy.before_model`. A predicate checks finite budget, one completed tool call, capacity beyond the submit reserve, and a per-run state flag. The nudge is added to the copied messages for one model call; no report, claim, tool argument, parser, scorer, or frozen file changes.

**Tech Stack:** Python 3.12+, existing middleware API, pytest, PowerShell, `run_practice.py`, `selfeval.py`, and `verify.py`.

## Global Constraints

- Modify only `harness/layers/budget_policy.py`; do not modify or create tests.
- Do not change `arena/`, `data/`, scorer, parser, trace protocol, `harness/agent.py`, `harness/middleware.py`, or `MAX_STEPS = 40`.
- Do not hard-code `brief_id`, answers, `doc_id`, `required_facts`, or `Doc.tags`.
- Do not change model-generated `claim["text"]`.
- Do not mutate `messages`, wrap hooks in `try/except`, call model/tools outside the harness, or retain cross-brief instance state.
- Preserve `.env`, existing `runs/`, and unrelated working-tree changes.
- Do not commit implementation automatically because `budget_policy.py` contains the user's uncommitted lab solution.
- Roll back only the retrieval-nudge delta if any acceptance threshold fails.

---

### Task 1: Characterize the missing behavior

**Files:**
- Read: `harness/layers/budget_policy.py:72`
- Read: `arena/model.py:420`
- Modify: none
- Test: inline Python assertion; no test file

**Interfaces:**
- Consumes: `BudgetPolicy.before_model(ctx, messages) -> list[dict]`, `ctx.state`, `ctx.max_tool_calls`, and `ctx.tools.calls`.
- Produces: a failing check proving no retrieval nudge exists yet.

- [ ] **Step 1: Run the pre-change check**

```powershell
@'
from types import SimpleNamespace
from harness.layers.budget_policy import BudgetPolicy

policy = BudgetPolicy()
ctx = SimpleNamespace(state={}, max_tool_calls=8, tools=SimpleNamespace(calls=1))
messages = [
    {"role": "user", "content": "Câu hỏi gốc"},
    {"role": "assistant", "content": "ACTION: search"},
    {"role": "user", "content": "OBSERVATION: kết quả ban đầu"},
]
result = policy.before_model(ctx, messages)
assert len(result) == len(messages) + 1
'@ | .\.venv\Scripts\python.exe -
```

Expected: FAIL at the final assertion.

- [ ] **Step 2: Capture the pre-change scoped diff**

```powershell
git -c safe.directory='D:/AI_In_Action/Codelabs/K3_Day16_2A202601883_BuiThoAn' diff -- harness/layers/budget_policy.py
```

Expected: the user's completed baseline implementation is present, with no retrieval-nudge symbols.

### Task 2: Implement the one-turn nudge

**Files:**
- Modify: `harness/layers/budget_policy.py:72`
- Test: inline assertions and existing tests only

**Interfaces:**
- Consumes: `_spent(ctx)`, `ctx.state`, `ctx.max_tool_calls`, `ctx.tools.calls`, and `self.reserve`.
- Produces: `RETRIEVAL_NUDGE_STATE_KEY`, `RETRIEVAL_NUDGE`, `_should_nudge_retrieval(ctx) -> bool`, and an updated `before_model`.

- [ ] **Step 1: Add constants beside `NUDGE`**

```python
RETRIEVAL_NUDGE_STATE_KEY = "budget_policy_retrieval_nudge_sent"

RETRIEVAL_NUDGE = (
    "Hãy đối chiếu câu hỏi với bằng chứng đã đọc. Nếu chưa có câu trả lời trực tiếp, "
    "hãy diễn đạt lại truy vấn bằng thuật ngữ nội bộ xuất hiện trong câu hỏi hoặc "
    "bằng chứng, không lặp lại truy vấn hay tài liệu đã đọc. Chỉ abstain sau khi đã "
    "search và fetch mà vẫn không có đủ căn cứ."
)
```

- [ ] **Step 2: Add the predicate below `_spent`**

```python
    def _should_nudge_retrieval(self, ctx) -> bool:
        state = getattr(ctx, "state", None)
        limit = getattr(ctx, "max_tool_calls", None)
        calls = getattr(getattr(ctx, "tools", None), "calls", 0)
        if not isinstance(state, dict) or limit is None or calls < 1:
            return False
        if calls >= limit - self.reserve:
            return False
        return not bool(state.get(RETRIEVAL_NUDGE_STATE_KEY))
```

- [ ] **Step 3: Preserve finalize priority in `before_model`**

```python
    def before_model(self, ctx, messages):
        if self._spent(ctx):
            return messages + [{"role": "user", "content": NUDGE}]
        if not self._should_nudge_retrieval(ctx):
            return messages
        ctx.state[RETRIEVAL_NUDGE_STATE_KEY] = True
        return messages + [{"role": "user", "content": RETRIEVAL_NUDGE}]
```

- [ ] **Step 4: Run focused contract assertions**

```powershell
@'
from types import SimpleNamespace
from arena.model import FINALIZE_SENTINEL
from harness.layers.budget_policy import BudgetPolicy, NUDGE, RETRIEVAL_NUDGE, RETRIEVAL_NUDGE_STATE_KEY

def make_ctx(calls, limit=8, state=None):
    return SimpleNamespace(state={} if state is None else state,
                           max_tool_calls=limit,
                           tools=SimpleNamespace(calls=calls))

messages = [
    {"role": "user", "content": "Câu hỏi gốc"},
    {"role": "assistant", "content": "ACTION: search"},
    {"role": "user", "content": "OBSERVATION: kết quả ban đầu"},
]
policy = BudgetPolicy()

turn_zero = make_ctx(0)
assert policy.before_model(turn_zero, messages[:1]) == messages[:1]

active = make_ctx(1)
first = policy.before_model(active, messages)
assert first == messages + [{"role": "user", "content": RETRIEVAL_NUDGE}]
assert first is not messages
assert active.state[RETRIEVAL_NUDGE_STATE_KEY] is True
assert FINALIZE_SENTINEL not in RETRIEVAL_NUDGE
assert policy.before_model(active, messages) == messages

finalizing = make_ctx(7)
final = policy.before_model(finalizing, messages)
assert final == messages + [{"role": "user", "content": NUDGE}]
assert FINALIZE_SENTINEL in final[-1]["content"]

assert policy.before_model(make_ctx(1, limit=None), messages) == messages
invalid = make_ctx(1)
invalid.state = object()
assert policy.before_model(invalid, messages) == messages
assert messages[-1]["content"] == "OBSERVATION: kết quả ban đầu"
'@ | .\.venv\Scripts\python.exe -
```

Expected: PASS with no output.

- [ ] **Step 5: Run focused existing tests**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe -m pytest -q tests\test_layers_stubs.py tests\test_middleware.py tests\test_model.py
```

Expected: all selected tests pass.

### Task 3: Verify repository and mock behavior

**Files:**
- Modify: none
- Generate: `runs/stability_mock_candidate.json`, expected to be ignored

**Interfaces:**
- Consumes: updated `BudgetPolicy`, existing runner, scorer, and tests.
- Produces: test, frozen-file, mock-score, safety, provenance, and trace evidence.

- [ ] **Step 1: Run all tests**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe -m pytest -q
```

Expected: all tests pass.

- [ ] **Step 2: Verify frozen files**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe scripts\verify.py
```

Expected: verification passes with no frozen-file modification.

- [ ] **Step 3: Run mock full stack**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe scripts\run_practice.py --out runs\stability_mock_candidate.json
```

Expected: mean at least 81.71, all trace gates pass, and no safety regression.

- [ ] **Step 4: Diagnose mock candidate**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe scripts\selfeval.py --run runs\stability_mock_candidate.json
```

Expected: no new provenance, hallucination, citation, trace, or safety failure attributable to the nudge.

- [ ] **Step 5: Inspect scope**

```powershell
git -c safe.directory='D:/AI_In_Action/Codelabs/K3_Day16_2A202601883_BuiThoAn' status --short
git -c safe.directory='D:/AI_In_Action/Codelabs/K3_Day16_2A202601883_BuiThoAn' diff --check
git -c safe.directory='D:/AI_In_Action/Codelabs/K3_Day16_2A202601883_BuiThoAn' diff --name-only
```

Expected: implementation delta is confined to `harness/layers/budget_policy.py`; unrelated user changes remain untouched.

### Task 4: Run one real-model comparison

**Files:**
- Modify: none unless rollback removes only the retrieval-nudge delta
- Generate: `runs/stability_real_candidate.json`, expected to be ignored

**Interfaces:**
- Consumes: configured real-model environment without printing secrets and baseline thresholds mean 42.13, minimum 26.67, zero-grounding count 6.
- Produces: one nine-brief comparison and a keep-or-rollback decision.

- [ ] **Step 1: Run the agreed real-model evaluation**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe scripts\run_practice.py --model real --prompt-addendum --out runs\stability_real_candidate.json
```

Expected: all nine briefs complete without exposing credentials.

- [ ] **Step 2: Diagnose the real candidate**

```powershell
$env:PYTHONIOENCODING='utf-8'
.\.venv\Scripts\python.exe scripts\selfeval.py --run runs\stability_real_candidate.json
```

Expected: diagnostics exist for all nine briefs and trace gates pass.

- [ ] **Step 3: Compute acceptance metrics**

```powershell
$candidate = Get-Content -Raw -Encoding UTF8 runs\stability_real_candidate.json | ConvertFrom-Json
$mean = [double]$candidate.mean_total
$minimum = [double](($candidate.runs | Measure-Object -Property total -Minimum).Minimum)
$zeroGrounding = @($candidate.runs | Where-Object { [double]$_.grounding -eq 0.0 }).Count
[pscustomobject]@{
    mean = [math]::Round($mean, 2)
    minimum = [math]::Round($minimum, 2)
    zero_grounding = $zeroGrounding
    mean_pass = $mean -gt 42.13
    minimum_pass = $minimum -gt 26.67
    zero_grounding_pass = $zeroGrounding -lt 6
} | Format-List
```

Expected to keep the change: all three pass fields are `True`.

- [ ] **Step 4: Enforce rollback criteria**

If all checks pass, keep the retrieval-nudge delta. If any check fails, use `apply_patch` to remove only the two retrieval constants, `_should_nudge_retrieval`, and the retrieval branch in `before_model`; restore the original completed `before_model`:

```python
    def before_model(self, ctx, messages):
        if not self._spent(ctx):
            return messages
        return messages + [{"role": "user", "content": NUDGE}]
```

Do not use `git checkout` or `git reset`, because they would discard the user's pre-existing solution.

- [ ] **Step 5: Report without committing code**

Report mock mean, real mean, real minimum, zero-grounding count, tests, verify result, keep-or-rollback decision, and exact modified file. State that one real run is a regression sample, not statistical proof of long-term stability.
