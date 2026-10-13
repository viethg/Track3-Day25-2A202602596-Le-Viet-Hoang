Vì sao bạn cần học bài này (Why)
AICB-P2T3 · Ngày 25 · Circuit Breakers, Caching & Reliability for Production Agents

Bạn vào vai on-call của ShopAgent: bản v1 vừa sập vì một retry storm. Bạn thay nó bằng một client chịu lỗi, chứng minh bằng số rằng nó chịu được 6 "băng sự cố", rồi viết postmortem.
Agent chạy production không chết vì model kém, mà thường chết vì cách nó phản ứng khi provider có sự cố. Một provider LLM chậm, hết quota hay trả 429 là chuyện xảy ra hằng tháng. Điều quyết định bạn mất 3 phút hay mất cả giờ là code xung quanh lời gọi LLM.

Trong lab này, bản v1 của ShopAgent có 3 tầng retry chồng nhau (app × gateway × SDK = tối đa 36 attempt cho mỗi call), delay cố định, không breaker, không fallback. Tuần trước nó biến một sự cố 3 phút của provider thành 5 phút sập hoàn toàn: provider đã hồi phục nhưng vẫn nghẹt vì chính lượng retry của mình (metastable failure).

Ba lý do bài này đáng làm:

1. Retry sai còn nguy hiểm hơn không retry. Mỗi lần retry thiếu kiểm soát là thêm tải đúng lúc hệ thống yếu nhất.
2. Agent có side effect. Một lần hoàn tiền bị gọi hai lần hoặc bị mất là tiền thật, nên không thể "thử lại cho chắc".
3. Phải chứng minh bằng số. Nói "hệ thống chịu lỗi" là chưa đủ: bạn phải đo availability, p95, amplification, thời gian phục hồi và viết postmortem như một kỹ sư on-call thật.

-------
Sau bài này bạn làm được gì
Phân loại lỗi (429 rate limit ≠ 429 hết quota ≠ 529 ≠ 400 ≠ timeout) và phản ứng đúng cho từng loại.
Retry đúng cách: chỉ lỗi transient, full jitter, tôn trọng retry-after, có retry budget, chỉ một tầng.
Viết circuit breaker sliding-window an toàn dưới tải đồng thời: trip theo tỷ lệ lỗi, bắt call chậm, giới hạn probe ở HALF-OPEN.
Xây fallback ladder có kiểm tra tương thích: không gửi tool call sang model không hỗ trợ tool, biết spillover khi bị rate limit.
Làm workflow exactly-once: idempotency key ổn định, checkpoint đúng thời điểm, resume sau khi worker chết.
Dùng semantic cache có kiểm soát (scope, ngưỡng, TTL, invalidation) và stale-if-error, để câu hỏi FAQ vẫn được trả lời khi mọi provider sập mà không trả lời sai hay lộ dữ liệu người khác.
Viết burn-rate alerting đa cửa sổ và tính error budget để biết khi nào page, khi nào chỉ cần ticket, khi nào đóng băng release.
Viết postmortem blameless có số liệu từ một sự cố mô phỏng.
Lab độc lập, không cần API key, không cần Docker. Chỉ cần Python ≥ 3.10, pyyaml, pytest. Thời lượng: ~4 giờ (2 giờ core trên lớp: Bước 3–5; 2 giờ cho Bước 6–9, report hoàn thiện ở nhà nếu cần).

Tổng điểm 100 = 88 điểm tự động (make grade) + 12 điểm chấm tay (report).

Bước	Nội dung	Điểm
1	Fork starter, đặt tên repo, cài đặt	—
2	Hiểu môi trường mô phỏng và 6 băng sự cố	—
3	Part A — Phân loại lỗi và retry	12
4	Part B — Circuit breaker sliding-window	12
5	Part C — ResilientClient	30
6	Part G — Semantic cache và stale-if-error	10
7	Part D — Workflow hoàn tiền exactly-once	12
8	Part E — Burn-rate alerting và error budget	12
9	Part F — Report và postmortem	12
10	Kiểm tra bằng make grade và nộp bài	—

-------
Bước 1 — Fork starter, đặt tên repo, cài đặt
Repo starter: https://github.com/VinUni-AI20k/K4-Track03-Circuit-Breakers-Caching-Reliability

Việc cần làm

1. Đăng nhập GitHub, mở link repo trên, bấm Fork để tạo bản của bạn.
2. Đổi tên repo ngay (Settings → General → Repository name) theo quy tắc:
Track3-Day25-<MSV>-<Họ-Và-Tên>

<MSV>: mã số sinh viên.
<Họ-Và-Tên>: không dấu, mỗi từ viết hoa chữ cái đầu, nối bằng dấu -, không dùng khoảng trắng.
Ví dụ: Nguyễn Văn An, MSV 20260123 → Track3-Day25-20260123-Nguyen-Van-An
1. Repo phải ở chế độ public (fork từ repo public thì mặc định public, không chuyển sang private).
2. Clone về máy và cài đặt:
git clone https://github.com/<username>/Track3-Day25-20260123-Nguyen-Van-An.git
cd Track3-Day25-20260123-Nguyen-Van-An
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -e ".[dev]"                                 # hoặc tối thiểu: pip install pyyaml pytest
make test
Chép
Kiểm tra: make test chạy được. Ở starter, phần lớn test FAIL, đó là việc của bạn ở các bước sau. Máy không có make thì chạy trực tiếp: python -m pytest -q, python scripts/run_gameday.py --client both.

------
Bước 2 — Hiểu môi trường mô phỏng và 6 băng sự cố
Bối cảnh. ShopAgent xử lý yêu cầu hoàn tiền. Mỗi lượt user gồm 2 LLM call: tool (cần tool calling) và chat. User chờ tối đa 30 giây cho cả lượt. Có 4 deployment:

Deployment	Model	Tool calling	Chi phí/call	Năng lực	Rate limit
alpha (primary)	A-large	có	1,0	18 slot, ~1,2 s/req	20 rps
alpha-eu	A-large	có	1,0	16 slot, ~1,5 s/req	10 rps
beta	B-large	có	1,4	30 slot, ~1,8 s/req	16 rps
gamma-mini	C-mini	không	0,2	40 slot, ~0,6 s/req	30 rps
Bản v1 là NaiveClient (src/gameday_lab/naive_client.py): 3 tầng retry chồng nhau (tối đa 36 attempt/call), delay cố định, không breaker, không fallback.

Quy tắc mô phỏng (đọc kỹ, không sửa sim/)

Thời gian ảo. Simulator là discrete-event, 8–10 phút sự cố chạy trong ~0,3 giây và lặp lại được với cùng seed. Mọi hàm nhận now (giây ảo). Không dùng time.time() / time.sleep().
Provider là hàng đợi. Request chờ quá 10 s thì bị 529. Client timeout thì server vẫn xử lý tiếp request đó. Đây là cơ chế biến retry storm thành metastable failure.
Client điều khiển bằng sự kiện: simulator gọi start(call, now), rồi on_success(...) / on_error(...). Bạn trả về Send(provider, timeout_s, delay_s, tag) hoặc GiveUp(reason). Xem src/gameday_lab/sim/contracts.py.
Lỗi	status	code	retry_after
Hết rate limit	429	rate_limit_exceeded	có
Hết quota / credit	429	insufficient_quota	không
Quá tải / chết	529 / 503	overloaded_error / service_unavailable	không
Lỗi server	500	api_error	không
Request sai (bug của mình)	400	invalid_request_error	không
Client hết kiên nhẫn	0	client_timeout	không
Sáu băng sự cố

Băng	Chuyện gì xảy ra	Phải đạt
G1 rate-limit spike	Traffic ×2,5 trong 3 phút; alpha chỉ còn 12 rps	availability ≥ 98%, 0 vi phạm retry-after, amplification ≤ 1,6
G2 overload storm	alpha mất 75% GPU trong 3 phút	availability ≥ 95%, amplification ≤ 1,6, recovery ≤ 30 s, ≥ 80% traffic quay lại alpha
G3 brown-out	alpha trả 500 cho 30% request	availability ≥ 98%, retry_ratio ≤ 0,10
G4 slow region	alpha chậm thêm 5 s, không có lỗi	availability ≥ 97%, p95 ≤ 10 s, quay lại alpha sau sự cố
G5 bad deploy → quota	60% request malformed 2 phút; sau đó account A hết quota	0 retry lỗi 400, ≤ 10 failover không cần thiết, availability ≥ 78%
G6 provider outage	Cả provider A sập; beta hạ rate limit	availability ≥ 95%, 0 tool call gửi sang model không hỗ trợ tool
Giảng viên chấm bằng băng ẩn (lệch thời điểm, đổi cường độ) và nhiều seed. Đừng hard-code thời gian hay tên băng.

Việc cần làm: chạy make baseline để xem v1 thất bại trên cả 6 băng, ghi lại số liệu G2 để dùng cho postmortem (Bước 9).

Quy tắc sửa file

Không sửa: src/gameday_lab/sim/, naive_client.py, workflows/services.py, workflows/runner.py, workflows/naive_executor.py, slo/data.py, slo/scoring.py, configs/scenarios.yaml, configs/scenarios_cache.yaml, tests/. Grader dùng bản riêng của các file này.
Được sửa: các file có TODO, khối client: và cache: trong configs/gameday.yaml, thêm file/test của bạn.

-------
Bước 3 — Part A: Phân loại lỗi và retry (12 điểm · ~25 phút)
File: src/gameday_lab/errors.py, src/gameday_lab/retry.py · Test: tests/test_errors_retry.py

Việc cần làm

classify(), is_retryable(), counts_toward_breaker(), failover_helps(): phân biệt 429 rate limit ≠ 429 hết quota ≠ 529 ≠ 400 ≠ timeout.
FullJitterBackoff.delay(): full jitter; nếu có retry_after thì không bao giờ retry sớm hơn.
RetryBudget: token bucket. Mỗi request gửi lần đầu nạp ratio token, mỗi retry tiêu 1 token, hồi min_per_s token/giây.
Kiểm tra: python -m pytest tests/test_errors_retry.py -q — tất cả pass.

Lỗi hay gặp: retry lỗi 400; bỏ qua retry-after; lỗi 400 bị tính vào breaker (except Exception).

------
Bước 4 — Part B: Circuit breaker sliding-window (12 điểm · ~40 phút)
File: src/gameday_lab/breaker.py · Test: tests/test_breaker.py

Việc cần làm

Trip theo tỷ lệ lỗi trên window_size kết quả gần nhất, chỉ khi đã có min_calls kết quả.
Call thành công nhưng chậm (≥ slow_call_s) là dấu hiệu xấu → trip theo slow_rate_threshold.
HALF-OPEN: tối đa half_open_max_probes request đang bay; đủ half_open_successes → CLOSED; probe hỏng hoặc chậm → OPEN.
Mỗi request được cấp một Permit. Kết quả mang generation cũ (về sau khi state đã đổi) thì bỏ qua.
Kiểm tra: python -m pytest tests/test_breaker.py -q — tất cả pass.

Lỗi hay gặp: HALF-OPEN không giới hạn probe; không có slow-call detection (G4 sẽ lộ ra ở p95).

------
Bước 5 — Part C: ResilientClient (30 điểm · ~50 phút)
File: src/gameday_lab/client.py · Test: tests/test_client.py (6 điểm) + 6 băng × 4 điểm

Việc cần làm

ladder(task): thứ tự fallback, lọc theo khả năng: tool call chỉ được đi model có tool calling (không gửi sang gamma-mini).
start / on_success / on_error / on_cancel: ghép Part A và Part B. Gợi ý quyết định có trong docstring.
Chỉ một tầng retry (không chồng app × gateway × SDK như v1).
Tinh chỉnh khối client: trong configs/gameday.yaml và giải thích các tham số trong report.
Kiểm tra:

python -m pytest tests/test_client.py -q
make gameday     # client của bạn trên 6 băng
make compare     # naive vs của bạn, cạnh nhau — chạy lại cho tới khi 6/6 băng PASS
Chép
Lỗi hay gặp: timeout quá dài (30 s) làm G6 treo mới failover; fail over khi gặp 400 (G5); tool call sang gamma-mini (G6).

------
Bước 6 — Part G: Semantic cache và stale-if-error (10 điểm · ~40 phút)
Làm ngay sau Part C vì băng C1 dùng ResilientClient của bạn.
File: src/gameday_lab/cache.py · Test: tests/test_cache.py (4 điểm) + băng C1 (6 điểm)

60% lượt user là câu hỏi FAQ. Mỗi lượt FAQ là một call chat mang call.request (text, user_id, personal, prompt_version, embedding). Khi call thành công, simulator đặt câu trả lời vào call.answer. Client trả lời từ cache bằng action Serve(answer, stale).

Việc cần làm

SemanticCache — lookup / store / invalidate:
tìm theo cosine trong cùng scope (prompt_version) với ngưỡng similarity_threshold;
câu hỏi cá nhân (request.personal) không được cache;
entry còn tươi trong ttl_s, sau đó là cửa sổ stale max_stale_s;
invalidate theo source; tối đa max_entries (LRU).
CachingClient bọc ResilientClient: hit còn tươi → Serve; miss → gọi provider rồi lưu call.answer; provider bỏ cuộc (GiveUp) → trả câu cũ nếu còn trong cửa sổ stale (stale-if-error); webhook on_content_update(source) → invalidate.
Khối cache: trong configs/gameday.yaml ở starter cố ý chưa tốt. Chạy make cache, đọc wrong_answers_by_reason, chỉnh từng tham số.
Băng C1_faq_outage (make cache): t=150 chính sách hoàn tiền đổi (có webhook) · t=240–360 mọi provider sập · t=420 deploy prompt v2 · t=480 bảng phí ship đổi không báo trước. Đạt khi:

wrong_answers = 0
outdated_answers ≤ 15
faq_availability ≥ 93%
cache_hit_ratio ≥ 50%
Kiểm tra: python -m pytest tests/test_cache.py -q (làm được ngay với client giả), rồi make cache. Nếu Part C chưa xong thì vẫn lấy được 4 điểm test.

Lỗi hay gặp: ngưỡng similarity quá lỏng (0,80) → other_question; cache câu cá nhân → other_user; scope thiếu prompt_version → old_prompt; bỏ qua webhook → ignored_invalidation; TTL quá dài → outdated_answers; max_stale_s ngắn hơn sự cố → faq_availability thấp.

------
Bước 7 — Part D: Workflow hoàn tiền exactly-once (12 điểm · ~30 phút)
File: src/gameday_lab/workflows/executor.py · Test: tests/test_executor.py (4 điểm) + make workflows (8 điểm)

Workflow 5 bước: classify_request → lookup_order → issue_refund → send_email → write_summary. Payment có thể đã chạy nhưng response bị mất (ServiceTimeout), hoặc worker chết ngay sau khi tiền đã đi (WorkerCrash).

Việc cần làm

Idempotency key ổn định theo (session, step), không sinh ngẫu nhiên mỗi lần gọi.
Checkpoint vào DurableStore sau khi step thành công.
Resume từ checkpoint sau crash.
Đạt: 0 hoàn tiền trùng, 0 email trùng, 0 hoàn tiền bị mất, ≥ 99% hoàn tất, wasted_llm_per_crash ≤ 0,6.

Kiểm tra: python -m pytest tests/test_executor.py -q rồi make workflows.

Lỗi hay gặp: uuid4() làm key → hoàn tiền trùng; ghi checkpoint trước khi chạy step → hoàn tiền bị mất; bắt WorkerCrash trong executor.

-------
Bước 8 — Part E: Burn-rate alerting và error budget (12 điểm · ~30 phút)
File: src/gameday_lab/slo/alerting.py · Test: tests/test_alerting.py (4 điểm) + make slo (4 điểm) + dữ liệu ẩn (4 điểm)

SLO availability 99,5% / 30 ngày. Dữ liệu: data/sli_30d.csv (43.200 phút) + nhãn data/sli_30d_labels.json.

Việc cần làm

Luật đa cửa sổ: page 14,4× (1h & 5m), page 6× (6h & 30m), ticket 1× (3d & 6h). Dùng prefix sum để tính nhanh.
error_budget_report(): budget đã tiêu bao nhiêu, có phải đóng băng release không.
Đạt: page đúng mọi sự cố lớn trong thời hạn, ticket cho rò rỉ chậm (không page), 0 false page cho các blip.

Kiểm tra: python -m pytest tests/test_alerting.py -q rồi make slo.

Lỗi hay gặp: alert theo ngưỡng 5 phút đơn lẻ → false page trên blip.

-----------
Bước 9 — Part F: Report và postmortem (12 điểm · chấm tay)
File: reports/REPORT.md (copy từ reports/REPORT_TEMPLATE.md)

Việc cần làm

1. cp reports/REPORT_TEMPLATE.md reports/REPORT.md và điền đủ các mục, kể cả mục 8 (cache).
2. Trọng tâm: postmortem blameless cho băng G2 với client naive, có số liệu (make plot cần pip install -e ".[plot]"; CSV nằm trong reports/gameday/).
3. Thí nghiệm bật/tắt retry budget: giải thích kết quả. Thí nghiệm 2 (G6, đối chứng G4): xem availability và give_up_reasons trước p95.
4. Giải thích các tham số client: và cache: bạn đã chọn bằng số.
Kiểm tra: đối chiếu với docs/RUBRIC.md, không bỏ trống mục nào.

-------
Bước 10 — Kiểm tra bằng `make grade` và nộp bài
Chấm thử:

make grade       # 88 điểm tự động -> reports/scoreboard.json
Chép
Chuẩn bị repo

Thư mục reports/ là một phần bài nộp: hãy commit cả REPORT.md, hình và scoreboard.json. Nếu Git bỏ qua scoreboard.json, dùng git add -f reports/scoreboard.json.
make clean xóa hình và CSV trong reports/: đừng chạy sau khi đã chuẩn bị xong.
Ghi commit hash nộp bài vào dòng Commit: đầu reports/REPORT.md, và ghi họ tên + MSV ở dòng Họ tên / nhóm.
Nộp

1. Push lên nhánh main của repo đã đổi tên (Bước 1).
2. Kiểm tra repo mở được khi không đăng nhập (public) và tên đúng Track3-Day25-<MSV>-<Họ-Và-Tên>.
3. Nộp link repo lên kênh nộp bài của lớp. Ví dụ: https://github.com/<username>/Track3-Day25-20260123-Nguyen-Van-An
Grader chạy: pip install -e ".[dev]" && make grade trên máy sạch, bằng bản băng ẩn và bản sim/ riêng của giảng viên.

-------
Phụ lục
Rubric (100 điểm): A 12 · B 12 · C 30 · D 12 · E 12 · G 10 · F 12 (chấm tay).

FAQ nhanh

Sao availability ở G5 chỉ cần ≥ 78%? Request malformed thì không client nào cứu được. Việc đúng là không làm tệ thêm: không retry, không failover vô ích.
Sao faq_availability ở C1 không lên 100%? Trong 2 phút mọi provider sập, câu hỏi cá nhân không được trả lời từ cache nên luôn thất bại.
Đọc thêm

Brooker, Exponential Backoff And Jitter (AWS Architecture Blog, 2015).
Google SRE Book, Handling Overload; SRE Workbook ch. 5, Alerting on SLOs.
Bronson et al., Metastable Failures in Distributed Systems (HotOS 2021).
Stripe API docs, Idempotent requests.
RFC 5861, HTTP Cache-Control Extensions for Stale Content.
vCache: Verified Semantic Prompt Caching (arXiv:2502.03771).
-------
