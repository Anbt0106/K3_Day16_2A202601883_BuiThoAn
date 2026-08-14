# Thiết kế ổn định model thật cho Agent Arena

## Mục tiêu

Cải thiện đồng thời điểm trung bình và điểm thấp nhất của một lượt chạy real model trên chín public brief, đồng thời giữ nguyên chất lượng mock model và mọi ranh giới của bài lab.

Mốc so sánh hiện tại lấy từ `runs/real_model.json`:

- điểm trung bình: 42.13;
- điểm thấp nhất: 26.67;
- sáu trong chín brief có grounding bằng 0;
- trace gate qua cả chín brief;
- điểm mock full stack: 81.71.

## Phạm vi

Chỉ sửa `harness/layers/budget_policy.py`. Không sửa hoặc tạo test vì người dùng yêu cầu chỉ thay đổi các file layer của bài lab và bộ test hiện có đã kiểm tra các hợp đồng middleware, sentinel, ngân sách và mock model.

Không thay đổi:

- `arena/`, `data/`, scorer, parser hoặc trace protocol;
- `harness/agent.py`, `harness/middleware.py` hoặc `MAX_STEPS = 40`;
- `tests/`, `.env`, dữ liệu `runs/` đã có hoặc thay đổi không liên quan của người dùng;
- `brief_id`, đáp án, `doc_id`, `required_facts` hoặc `Doc.tags` theo cách hard-code;
- nội dung `claim["text"]` do model sinh;
- số lượng năm layer và thứ tự stack do runner quy định.

## Phương án được chọn

Mở rộng `BudgetPolicy.before_model` bằng một retrieval nudge tổng quát, chỉ xuất hiện trong đúng một model turn sau khi agent đã gọi ít nhất một tool.

Nudge yêu cầu model kiểm tra xem bằng chứng hiện có đã trực tiếp trả lời câu hỏi chưa. Nếu chưa, model phải diễn đạt lại truy vấn bằng thuật ngữ từ câu hỏi và tài liệu đã quan sát, không lặp truy vấn hoặc fetch cũ, và không abstain trước khi đã search rồi fetch.

Không can thiệp hoặc tự viết lại arguments của tool. Model vẫn là chủ thể chọn query và tool call, nhờ đó trace và provenance giữ nguyên.

## Điều kiện kích hoạt

Nudge chỉ được chèn khi đồng thời thỏa mãn:

1. `ctx.state` là dictionary.
2. `ctx.max_tool_calls` là một giới hạn hữu hạn.
3. `ctx.tools.calls >= 1`, bảo đảm trong lịch sử đã có assistant turn và tool observation.
4. Còn nhiều hơn phần ngân sách dành cho `submit`; nudge không được cướp lượt cuối.
5. Cờ riêng của `BudgetPolicy` chưa được đặt trong `ctx.state`.
6. Ngân sách chưa cạn theo `_spent(ctx)`; khi đã cạn, nudge finalize hiện có luôn được ưu tiên.

Nudge không kích hoạt ở turn zero. Điều này giữ nguyên câu hỏi mà `arena.model._first_user_content` khôi phục cho mock model.

## Nội dung và định dạng nudge

Nudge là một user message dùng trong bản sao message của đúng model turn hiện tại. Nó không chứa `FINALIZE_SENTINEL`, vì sentinel sẽ buộc mock model chốt FINAL ngay.

Nội dung chỉ nêu chiến lược tổng quát:

- đối chiếu câu hỏi với bằng chứng đã đọc;
- nếu chưa có câu trả lời trực tiếp, đổi truy vấn bằng thuật ngữ nội bộ xuất hiện trong câu hỏi hoặc bằng chứng;
- không lặp lại search/fetch;
- chỉ abstain sau khi đã search và fetch mà vẫn không có căn cứ.

Không chép dữ kiện từ public brief, không nêu mã tài liệu và không tự sinh claim.

## Luồng xử lý

1. `before_model` kiểm tra `_spent(ctx)` trước. Nếu đã cạn ngân sách, trả `messages + [NUDGE]` như hiện tại.
2. Nếu chưa cạn, gọi helper kiểm tra điều kiện retrieval nudge.
3. Nếu không đủ điều kiện, trả nguyên `messages`.
4. Nếu đủ điều kiện, đặt cờ trong `ctx.state` rồi trả `messages + [RETRIEVAL_NUDGE]`.
5. Vì message chỉ được thêm sau tool call đầu tiên, nó nằm sau assistant turn; mock model tiếp tục lấy câu hỏi gốc từ preamble.
6. `wrap_tool_call` và `Retry` tiếp tục bảo vệ ngân sách `submit` như hiện tại.

Không mutate `messages`, không bọc hook bằng `try/except`, không gọi model/tool ngoài harness và không lưu state trên instance dùng chung.

## Xử lý biên

- `ctx.state` không phải dictionary: không gửi retrieval nudge.
- `ctx.max_tool_calls is None`: không gửi retrieval nudge vì không thể chứng minh còn ngân sách.
- `ctx.tools.calls < 1`: không gửi, tránh làm đổi câu hỏi mock ở preamble.
- Chỉ còn phần reserve: dùng finalize policy hiện tại, không gửi retrieval nudge.
- Nudge đã gửi: không gửi lại trong cùng brief.

## Kiểm thử và tiêu chí chấp nhận

Thứ tự kiểm chứng:

1. Chạy test tập trung cho middleware/model/budget bằng bộ test hiện có.
2. Chạy toàn bộ `pytest`.
3. Chạy `scripts/verify.py` để xác nhận file đóng băng nguyên vẹn.
4. Chạy mock full stack một lần; điểm không được thấp hơn 81.71, trace gate qua cả chín brief và safety không giảm.
5. Chạy `selfeval.py` trên kết quả mock; không xuất hiện thêm `NOT_FROM_MODEL`, `NOT_SUBMITTED`, `HALLUCINATED`, `UNRETRIEVED` hoặc fabricated citation do thay đổi.
6. Chạy real model đúng một lần trên chín public brief với `--prompt-addendum`, theo yêu cầu người dùng.
7. So sánh ba chỉ số real model với baseline: mean lớn hơn 42.13, minimum lớn hơn 26.67 và số brief grounding bằng 0 ít hơn sáu.

Do chỉ chạy real model một lần, kết quả là kiểm tra hồi quy trên mẫu hiện tại, không phải bằng chứng thống kê về phương sai dài hạn.

## Điều kiện thoái lui

Loại bỏ riêng retrieval nudge và giữ nguyên lời giải năm layer hiện tại nếu xảy ra bất kỳ điều nào:

- test hoặc `verify.py` thất bại;
- mock model thấp hơn 81.71;
- trace, safety hoặc provenance của mock giảm;
- real-model mean không tăng;
- real-model minimum không tăng;
- số brief grounding bằng 0 không giảm;
- tool call vượt ngân sách.

Không chỉnh scorer, dữ liệu, model, prompt addendum trong `agent.py` hoặc public brief để làm kết quả đạt tiêu chí.
