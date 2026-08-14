# Premature Abstain Retry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cho model thật thêm đúng một cơ hội tạo ACTION search khi nó FINAL với `abstain=true` và không có claim, nhưng giữ nguyên đường mock và tự rollback nếu ba chỉ số real không cùng cải thiện.

**Architecture:** Bổ sung `wrap_model_call` trong `BudgetPolicy`. Hook giữ nguyên response đầu, chỉ gọi model lần hai khi lịch sử đã có retrieval, ngân sách còn đủ và retry chưa dùng; response hai chỉ được chấp nhận nếu parser xác nhận ACTION search.

**Tech Stack:** Python 3, middleware hiện có, `arena.model.parse_output`, helper chuẩn hóa công khai của scorer, pytest và practice runner.

## Global Constraints

- Chỉ sửa production file `harness/layers/budget_policy.py`.
- Không sửa `arena/`, `data/`, `tests/`, agent, middleware, scorer, parser, trace protocol, `.env`, `MAX_STEPS`, stack hoặc thứ tự layer.
- Không hard-code brief, tài liệu, fact, đáp án hoặc tags; không tạo hay sửa claim text.
- Tối đa một model retry mỗi brief và đúng một real-model evaluation chín brief.
- Chỉ giữ patch nếu mean, minimum và số brief grounding 0 đều cải thiện; nếu không, rollback riêng patch bằng `apply_patch`.
- Không commit implementation; các commit chỉ dành cho design/plan đã được duyệt.

---

### Task 1: Xác minh trace và khóa hành vi bằng test đỏ

**Files:**
- Inspect: `harness/agent.py`
- Inspect: `harness/middleware.py`
- Inspect: `arena/runner.py`
- Test: inline Python, không tạo hoặc sửa file test

**Interfaces:**
- Consumes: `Middleware.wrap_model_call(ctx, call, messages)` và `ModelResponse`.
- Produces: bằng chứng mỗi `call(messages)` vật lý tạo một `model_call`, cùng một characterization test thất bại trước implementation.

- [ ] **Step 1:** Chạy một agent tối giản có wrapper gọi inner model hai lần; xác nhận trace có hai event `model_call`. Dừng nếu accounting chỉ ghi một call.
- [ ] **Step 2:** Chạy inline test với `BudgetPolicy` hiện tại: response đầu là FINAL abstain không claim, response hai là ACTION search; kỳ vọng hiện tại chỉ gọi một lần và assertion thất bại.
- [ ] **Step 3:** Ghi lại chính xác lỗi đỏ để chứng minh test kiểm tra hành vi mới, không phải lỗi fixture.

### Task 2: Cài retry tối thiểu trong BudgetPolicy

**Files:**
- Modify: `harness/layers/budget_policy.py`

**Interfaces:**
- Consumes: `parse_output(normalized_text)`, `ctx.messages`, `ctx.state`, `ctx.tools.calls`, `ctx.max_tool_calls`.
- Produces: `BudgetPolicy.wrap_model_call` trả response đầu hoặc một ACTION search nguyên văn từ model thứ hai.

- [ ] **Step 1:** Thêm hằng state key và instruction yêu cầu đúng một ACTION search mới.
- [ ] **Step 2:** Thêm helper kiểm tra claim hợp lệ: `claims` phải là list và có ít nhất một dict chứa `text` và `doc_id` dạng chuỗi không rỗng.
- [ ] **Step 3:** Thêm helper đọc history bằng parser chuẩn hóa, chỉ công nhận ACTION `search` hoặc `fetch_doc` đã xuất hiện.
- [ ] **Step 4:** Implement `wrap_model_call`: gọi response đầu; chỉ kích hoạt khi FINAL, `abstain is True`, không claim hợp lệ, state là dict, chưa retry, đã retrieval và `tools.calls < max_tool_calls - reserve`.
- [ ] **Step 5:** Đánh dấu state trước lần gọi thứ hai; truyền bản sao messages cộng raw response đầu dưới role assistant và instruction dưới role user.
- [ ] **Step 6:** Chỉ trả response hai nếu parser chuẩn hóa cho `kind == "action"` và `tool == "search"`; mọi trường hợp khác trả nguyên response đầu.
- [ ] **Step 7:** Chạy lại inline characterization test; yêu cầu hai model call, response trả về là ACTION search và lần gọi tiếp theo không retry nữa.

### Task 3: Regression và mock gate

**Files:**
- Verify: `harness/layers/budget_policy.py`
- Output: ignored run artifact dưới `runs/`

**Interfaces:**
- Consumes: patch Task 2.
- Produces: bằng chứng focused tests, Windows regression, verify và mock baseline không suy giảm.

- [ ] **Step 1:** Chạy focused middleware/model tests; yêu cầu xanh.
- [ ] **Step 2:** Chạy Windows-compatible regression; yêu cầu ít nhất `738 passed, 1 skipped, 14 deselected`.
- [ ] **Step 3:** Chạy `scripts/verify.py`; chấp nhận duy nhất MD5 CRLF khi Git blob hashes vẫn khớp expected.
- [ ] **Step 4:** Chạy practice mock; yêu cầu mean >= 81.71, không gate lỗi, safety 30/30 và không provenance verdict xấu.

### Task 4: Một lần kiểm chứng model thật và quyết định giữ/rollback

**Files:**
- Compare: `runs/real_model.json`
- Create ignored artifact: `runs/premature_abstain_retry_real.json`
- Potential rollback: `harness/layers/budget_policy.py`

**Interfaces:**
- Consumes: candidate đã qua Task 3 và ba metric baseline.
- Produces: quyết định nhị phân dựa trên mean, minimum và grounding-zero count.

- [ ] **Step 1:** Nạp ba biến provider từ `.env` vào process mà không in secret; chạy đúng một lệnh `run_practice.py --model real --prompt-addendum` cho chín brief.
- [ ] **Step 2:** Tính candidate mean, minimum và số brief grounding bằng 0 bằng script đọc JSON; không chạy lại model.
- [ ] **Step 3:** Giữ patch chỉ khi cả ba bất đẳng thức nghiêm ngặt đều đúng; nếu một tiêu chí không đạt, dùng `apply_patch` gỡ đúng model-retry delta.
- [ ] **Step 4:** Sau rollback (nếu có), chạy focused test và mock lần cuối; báo kết quả thực tế, không commit implementation.
