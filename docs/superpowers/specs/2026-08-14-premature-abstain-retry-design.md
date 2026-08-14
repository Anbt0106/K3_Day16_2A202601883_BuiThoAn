# Thiết kế retry khi model thật abstain quá sớm

## Mục tiêu

Giảm số brief grounding bằng 0, tăng điểm thấp nhất và điểm trung bình của model thật bằng cách cho model thêm đúng một cơ hội tiếp tục retrieval khi nó chuẩn bị kết luận thiếu căn cứ quá sớm.

Lời giải năm layer hiện tại phải được giữ nguyên nếu thử nghiệm không cải thiện đồng thời ba chỉ số. Mock baseline phải giữ 81.71.

## Phạm vi

Chỉ xem xét sửa `harness/layers/budget_policy.py`. Không sửa `arena/`, `data/`, `tests/`, `harness/agent.py`, `harness/middleware.py`, scorer, parser, trace protocol, `.env`, `MAX_STEPS`, năm-layer stack hoặc thứ tự layer.

Không hard-code public brief, `brief_id`, `doc_id`, `required_facts`, đáp án hoặc `Doc.tags`. Không viết, sửa hoặc bổ sung `claim["text"]`.

## Cơ chế

`BudgetPolicy.wrap_model_call` gọi model bình thường và kiểm tra response bằng `arena.model.parse_output` sau bước chuẩn hóa công khai mà harness đang dùng, nếu có thể nhập lại đúng helper mà không sửa file khác.

Response được xem là premature abstain khi:

1. parser nhận được FINAL;
2. report đồng thời có `abstain=true` và không có claim hợp lệ;
3. agent đã gọi ít nhất một tool;
4. lịch sử cho thấy đã có search/fetch nhưng vẫn còn ngân sách tool ngoài phần reserve cho submit;
5. retry model-level chưa từng chạy trong brief hiện tại.

Nếu đủ điều kiện, layer gọi model thêm đúng một lần với bản sao messages cộng một user instruction yêu cầu bỏ kết luận sớm và trả đúng một ACTION search bằng truy vấn mới dựa trên thuật ngữ đã quan sát. Trạng thái đã retry được lưu trong `ctx.state`.

## Bảo toàn response và provenance

- Không dùng `after_model` để tự tạo ACTION vì trace đã đóng dấu raw model output trước hook đó.
- Không sửa text của FINAL hoặc claim.
- Nếu response thứ hai lỗi, không parse được, hoặc vẫn là FINAL thay vì ACTION, trả response đầu tiên.
- Chỉ trả response thứ hai khi parser xác nhận đó là ACTION search hợp lệ.
- Mock path phải không kích hoạt vì FINAL mock trước `critic` có `abstain=false` và claims không rỗng.

## Feasibility gate cho trace

Trước khi triển khai phải xác minh một `wrap_model_call` gọi `call(messages)` hai lần có tạo trace/model-call accounting hợp lệ hay không.

- Nếu mỗi physical call được runner ghi riêng và trace vẫn hợp lệ, tiếp tục.
- Nếu chỉ response cuối được ghi hoặc token/model-call accounting mất một request, dừng thiết kế và không sửa code.
- Không được sửa runner hoặc tự phát event để vượt gate.

## Ngân sách và lỗi

- Tối đa một model retry mỗi brief.
- Không retry khi `ctx.max_tool_calls is None`, state không phải dictionary hoặc ngân sách chỉ còn reserve.
- Retry không trực tiếp gọi tool; model phải tự trả ACTION và agent tiếp tục vòng chuẩn.
- Không bọc hook bằng `try/except`; chỉ dùng kiểm tra kiểu và parser contract hiện có.

## Kiểm chứng

1. Characterization test phải thất bại trước khi viết production code.
2. Focused middleware/model tests phải xanh, ngoại trừ lỗi baseline Windows đã chứng minh.
3. Full regression phù hợp Windows phải giữ mốc 738 passed, 1 skipped, 14 deselected hoặc tốt hơn.
4. `verify.py` phải giữ 20/21; lỗi duy nhất được phép là MD5 do CRLF, và Git blob frozen phải khớp hash chuẩn.
5. Mock mean phải ít nhất 81.71, mọi gate qua, safety 30/30 và không thêm provenance verdict xấu.
6. Chạy đúng một real-model evaluation gồm chín brief với `--prompt-addendum`.
7. So với baseline real gần nhất, candidate phải đồng thời có mean cao hơn, minimum cao hơn và số brief grounding 0 thấp hơn.

## Rollback

Nếu feasibility gate không qua hoặc bất kỳ tiêu chí kiểm chứng nào không đạt, gỡ riêng model-retry delta bằng `apply_patch`. Không dùng `git checkout`/`reset`, không sửa file bị cấm và không giữ một thay đổi chỉ làm mean tăng nhưng minimum hoặc grounding-0 không cải thiện.
