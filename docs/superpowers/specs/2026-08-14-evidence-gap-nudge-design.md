# Thiết kế Evidence-Gap Nudge cho Agent Arena

## Mục tiêu

Cải thiện khả năng truy xuất bằng chứng trên private brief và model thật mà không tối ưu cứng cho chín public brief. Thay đổi chỉ được giữ nếu không làm giảm safety, provenance, trace conformance hoặc hiệu quả ngân sách.

Mốc hiện tại là 81.71/100 trên mock model. `selfeval.py` cho thấy phần điểm thiếu chủ yếu đến từ dữ kiện bắt buộc chưa được truy xuất, đặc biệt ở `pub-08` và `pub-09`; citation, injection và honesty hiện không phải nguyên nhân chính.

## Phạm vi

Chỉ xem xét thay đổi trong `harness/layers/budget_policy.py` và các test hiện có liên quan trực tiếp nếu thực sự cần cập nhật. Không thay đổi:

- `arena/`, `data/`, scorer, parser hoặc trace protocol;
- `harness/agent.py`, `harness/middleware.py` hoặc `MAX_STEPS = 40`;
- `brief_id`, đáp án, `doc_id` hay `Doc.tags` theo cách hard-code;
- nội dung `claim["text"]` do model sinh;
- `.env`, dữ liệu trong `runs/` và thay đổi không liên quan của người dùng.

Nếu test hiện có đã đủ bao phủ, không tạo hoặc sửa test chỉ để hợp thức hóa hành vi mới.

## Phương án được chọn

Mở rộng `BudgetPolicy` bằng một lời nhắc truy xuất bổ sung có điều kiện. Đây là control-flow policy, không phải lớp tự sinh bằng chứng. Nó chỉ nhắc model tìm thêm bằng chứng khi còn ngân sách và chưa gửi lời nhắc trong lượt chạy.

Không thêm layer thứ sáu vì runner và yêu cầu bài lab xác định stack gồm năm layer. Không đặt logic retrieval vào `critic` hoặc `citation_checker`, vì hai layer đó chạy hậu kiểm report và có trách nhiệm riêng biệt.

## Điều kiện kích hoạt

Evidence-gap nudge chỉ được gửi khi đồng thời thỏa mãn:

1. Brief có giới hạn tool call hữu hạn.
2. Agent đã quan sát một số nội dung nhưng chưa đến ngưỡng finalize.
3. Còn đủ ngân sách cho tối thiểu một lượt tool hữu ích và một lượt `submit` dự trữ.
4. Lời nhắc chưa từng được gửi trong lượt chạy hiện tại.
5. Lịch sử chưa cho thấy model đã bắt đầu chốt `FINAL`.

Trạng thái "đã nhắc" được lưu trong `ctx.state` bằng khóa riêng, tránh thuộc tính toàn cục và tránh rò rỉ giữa các brief.

## Nội dung lời nhắc

Lời nhắc phải mang tính tổng quát: yêu cầu model kiểm tra xem bằng chứng hiện có đã trả lời đầy đủ câu hỏi chưa; nếu chưa, thực hiện một truy vấn hẹp hơn dựa trên phần thông tin còn thiếu.

Lời nhắc không được chứa:

- đáp án hoặc dữ kiện từ public brief;
- `brief_id`, `doc_id`, tên tài liệu cố định hoặc `Doc.tags`;
- câu chữ lấy từ `required_facts`;
- nội dung có thể bị scorer xem là claim do middleware tự viết.

Trước khi triển khai phải xác minh cách `arena.model._first_user_content` xử lý message bổ sung. Nếu một lời nhắc user thông thường làm mock model thay đổi câu hỏi gốc, thiết kế sẽ không dùng cách đó. Không tái sử dụng `FINALIZE_SENTINEL` cho lời nhắc retrieval nếu sentinel làm model chốt FINAL; khi đó phương án thử nghiệm phải dừng thay vì thay parser hoặc model.

## Luồng xử lý

1. `before_model` kiểm tra điều kiện finalize hiện tại; nếu ngân sách đã cạn thì giữ nguyên hành vi `NUDGE` hiện có.
2. Nếu chưa cạn, kiểm tra điều kiện evidence-gap.
3. Nếu đủ điều kiện, trả về một bản sao `messages + [nudge]` và đánh dấu trong `ctx.state`.
4. Nếu không đủ điều kiện, trả nguyên danh sách message.
5. `wrap_tool_call` và `Retry` tiếp tục bảo vệ phần ngân sách dành cho `submit` như hiện tại.

Không mutate trực tiếp `messages`, không nuốt exception và không gọi model/tool ngoài harness.

## Xử lý lỗi và cơ chế thoái lui

- Nếu `ctx.state` không phải dictionary, không gửi evidence-gap nudge.
- Nếu `ctx.max_tool_calls` là `None`, không suy diễn ngân sách và không kích hoạt thử nghiệm.
- Nếu không thể xác định an toàn rằng lời nhắc không đổi câu hỏi mock, không triển khai lời nhắc.
- Nếu bất kỳ test, trace gate, safety hoặc provenance nào giảm, hoàn nguyên riêng thay đổi evidence-gap; giữ nguyên năm phần TODO đã hoàn thiện.

## Kiểm thử và tiêu chí chấp nhận

Chạy các kiểm tra sau trên trạng thái trước và sau thay đổi:

1. Toàn bộ `pytest` phải xanh.
2. `scripts/verify.py` phải xác nhận file đóng băng nguyên vẹn.
3. Mock full stack không thấp hơn 81.71 và trace gate phải qua toàn bộ brief.
4. Safety không thấp hơn 30/30 trên bất kỳ public brief nào đang đạt mức đó.
5. Không xuất hiện thêm `NOT_FROM_MODEL`, `NOT_SUBMITTED`, `HALLUCINATED`, `UNRETRIEVED` hoặc fabricated citation.
6. Leave-one-out chứng minh thay đổi có tác dụng đo được; nếu không có tác dụng thì loại bỏ.
7. Chạy nhiều seed/flaky trial để kiểm tra phương sai và ngân sách, không chỉ chọn một lượt đẹp.
8. Model thật được dùng như phép kiểm tra bổ sung; không ghi API key vào lệnh, log hoặc source. Không khẳng định cải thiện nếu chỉ có một lượt chạy biến động.

Thay đổi được chấp nhận khi tăng retrieval/grounding một cách lặp lại mà không đánh đổi safety, provenance hoặc giới hạn tool. Nếu không đạt đầy đủ tiêu chí, kết luận thiết kế thực nghiệm không phù hợp và giữ nguyên lời giải tham chiếu 81.71.

## Rủi ro

- Mock model có thể hiểu message user cuối cùng là câu hỏi chính và truy xuất sai hoàn toàn.
- Một lượt nudge có thể tiêu ngân sách cần cho fetch hoặc submit.
- Model thật có thể trả `FINAL` sớm hoặc diễn giải lời nhắc không nhất quán.
- Điểm mock tăng trên một seed nhưng phương sai tăng hoặc private generalization giảm.

Các rủi ro này được kiểm soát bằng kích hoạt một lần, dự trữ ngân sách, kiểm tra provenance và tiêu chí thoái lui bắt buộc.
