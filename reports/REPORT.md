# Lab 25B — Report

**Họ tên / nhóm:** Lê Việt Hoàng - 2A202602596  **Commit:** 18cc181d9577bd2e4c45ab64eb6e23da86c20030  **Điểm tự động (`make grade`):** 88 / 88 (84.0 / 84 public + 4 hidden)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G4 và G6: P50 latency bình thường của provider là 1.2–1.8 s. Đặt 8.0 s đủ dài cho call hợp lệ nhưng đủ ngắn để kịp failover trước khi chạm turn deadline (30 s). Nếu đặt 30 s như Thí nghiệm 2 sẽ khiến 109 request bị quá hạn turn deadline (`give_up: deadline`). |
| max_attempts_per_provider | 2 | Cho phép thử lại đúng 1 lần trên cùng provider khi gặp lỗi tạm thời (transient). Dựa trên G1 và G2: ngăn chặn khuếch đại bão retry; nếu sau 1 retry vẫn lỗi thì failover ngay sang provider phụ. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Tránh thundering herd; cap 4.0 s đảm bảo retry vẫn kịp thời gian deadline của lượt turn (30 s). |
| retry_budget_ratio | 0.1 | Dựa trên G3: giữ `retry_ratio` ở mức 0.034 (đạt yêu cầu ≤ 0.10). Giới hạn tổng số lần retry không vượt quá 10% lưu lượng gốc, chặn đứng retry storm. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ 20 kết quả đủ nhạy để bắt sự cố; `min_calls: 10` ngăn ngừa việc breaker ngắt mạch nhầm khi tải thấp (ví dụ ban đêm chỉ có 1-2 call lỗi ngẫu nhiên). |
| breaker.failure_rate_threshold | 0.5 | Khi tỷ lệ lỗi nghiêm trọng đạt 50% trong cửa sổ trượt, mở mạch (OPEN) ngay để cắt tải và bảo vệ provider. |
| breaker.slow_call_s | 5.0 | Dựa trên G4: alpha bị trễ thêm 5 s (median lên ~6.2 s). Ngưỡng 5.0 s nhận diện cuộc gọi chậm bất thường, chuyển mạch giúp P95 đạt 5.562 s (thay vì 13.764 s ở client naive). |
| breaker.slow_rate_threshold | 0.6 | Mở mạch khi 60% cuộc gọi trong cửa sổ bị chậm, dù không có mã lỗi HTTP nào trả về. |
| breaker.open_duration_s | 10.0 | Cách ly provider trong 10 s để hàng đợi xử lý giải phóng tài nguyên trước khi bắt đầu thăm dò lại. |
| breaker.half_open_max_probes | 2 | Chỉ cho phép tối đa 2 request probe bay đồng thời trong HALF-OPEN, tránh dồn tải đột ngột (thundering herd) lên provider đang hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
                        [Nhận ProviderError]
                                 │
                     Phân loại lỗi: classify(err)
                                 │
       ┌─────────────────────────┼─────────────────────────┐
       ▼                         ▼                         ▼
[CLIENT_ERROR (400)]   [QUOTA_EXHAUSTED (429)]   [RATE_LIMITED (429)]
 (Lỗi cú pháp/bug)       (Hết credit/billing)       (Vượt rps tenant)
       │                         │                         │
 Breaker: release          Breaker: release          Breaker: release
 Retry: KHÔNG              Retry: KHÔNG              Retry: KHÔNG
 Failover: KHÔNG           Failover: CÓ              Failover: CÓ ngay
 Cooldown: Không           Cooldown: quota_cooldown  Cooldown: retry_after
       │                         │                         │
       ▼                         ▼                         ▼
 GiveUp("client_error")    Chuyển provider tiếp theo  Chuyển provider tiếp theo
                                 │
                                 ▼
              [OVERLOADED (529/503) / SERVER_ERROR (500) / TIMEOUT (0)]
                                 │
                      Breaker: on_failure()
                                 │
           ┌─────────────────────┴─────────────────────┐
           ▼                                           ▼
[tries < max_attempts VÀ                    [Hết lượt retry HOẶC
 budget cho phép VÀ                          hết budget HOẶC
 còn đủ thời gian deadline]                  hết deadline HOẶC breaker OPEN]
           │                                           │
 Thử lại cùng provider                      Failover sang provider tiếp theo
 (Delay = FullJitterBackoff)                (Reset tries = 0, delay = 0)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**  
   Giả lập breaker bị vô hiệu hóa (`breaker.min_calls: 1000000000`, `max_attempts_per_provider: 5`):
   - **(a) Khi BẬT Retry Budget:** Băng G2 đạt `availability = 0.9987`, `amplification = 1.383`, `retry_ratio = 0.058`, `recovery_s = 0.0`. Băng G3 đạt `availability = 1.0`, `amplification = 1.136`, `retry_ratio = 0.068`.
   - **(b) Khi TẮT Retry Budget:** Băng G2 bị khuếch đại bão tải với `amplification = 1.672`, `retry_ratio = 0.491` (gấp 8.5 lần!), `recovery_s = 30.0 s` (hồi phục chậm vì hàng đợi nghẹt), `availability` rớt xuống `0.9649`. Băng G3 có `retry_ratio = 0.162` (gấp 2.4 lần).
   - *Kết luận:* Ngay cả khi breaker hoạt động bình thường, hệ thống vẫn bắt buộc phải có cả hai lớp phòng vệ: Circuit breaker bảo vệ theo tình trạng tổng thể của provider để ngắt mạch và failover, còn Retry budget kiểm soát lưu lượng retry ở cấp độ client, ngăn không cho hàng nghìn request đồng thời cùng retry làm sập ngược hệ thống.

2. **`attempt_timeout_s: 30`.**  
   Khi nâng attempt timeout lên 30 s:
   - Ở băng G4, circuit breaker vẫn phát hiện cuộc gọi chậm (`slow_call_s = 5.0 s`) và trip kịp thời nên `availability = 1.0`, `P95 = 5.683 s`.
   - Tuy nhiên ở băng G6 (provider A sập hoàn toàn và treo): Lượt gọi đầu tiên tới `alpha-eu` bị treo và giữ chân client tới gần hết turn deadline (30 s). Hậu quả là client không còn đủ thời gian để failover sang `beta` hay `gamma-mini`, khiến `availability` rớt thảm hại xuống `0.9488` (trượt tiêu chuẩn ≥ 0.95) với 109 request thất bại vì quá hạn (`give_up: deadline`).
   - *Kết luận:* Timeout "đúng" không được dựa vào turn deadline tối đa của người dùng, mà phải dựa trên P99 latency thông thường của provider cộng với một biên độ nhỏ (buffer ~2-3x median), vừa đủ để request hợp lệ hoàn thành nhưng đủ ngắn để kịp thời failover sang provider dự phòng trước khi hết giờ.

3. **Bỏ lọc tool calling khỏi `ladder()`.**  
   Khi bỏ lọc khả năng tool calling và cho phép mọi task đi qua tất cả các deployment ở băng G6:
   - Xuất hiện `incompatible_routes = 258`, khiến `availability` tụt từ `1.0` xuống `0.8918`.
   - *Góc nhìn Product Manager:* "Hệ thống đã định tuyến nhầm 258 yêu cầu tra cứu/hoàn tiền của khách hàng sang model giá rẻ `gamma-mini` — model không hề có khả năng kích hoạt công cụ kế toán và kiểm tra đơn hàng. Kết quả là hơn 10% khách hàng nhận phản hồi hỏng, bot không thể thực hiện hoàn tiền khiến khách hàng bức xúc khiếu nại, phá hỏng tỷ lệ chuyển đổi dịch vụ."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trọng tâm: Vì sao chỉ idempotency key là **chưa đủ**, và chỉ checkpoint cũng **chưa đủ**?
- Nếu chỉ có **Idempotency Key** mà không có **Checkpoint**: Khi worker crash và restart, nó phải chạy lại toàn bộ workflow từ đầu (bước 1 classify, bước 2 lookup, bước 3 refund...). Dù cổng thanh toán không trừ tiền lần 2 nhờ key, ta vẫn lãng phí rất nhiều chi phí gọi LLM và thời gian xử lý (`wasted_llm_per_crash > 1.1`).
- Nếu chỉ có **Checkpoint** mà không có **Idempotency Key**: Nếu worker bị crash đúng vào khoảnh khắc ngay sau khi cổng thanh toán đã trừ tiền nhưng trước khi checkpoint kịp lưu, worker mới sẽ thực hiện lại lệnh thanh toán với một mã mới, dẫn tới thảm họa thanh toán trùng (`duplicate refunds`).
- **Nếu ghi checkpoint TRƯỚC khi gọi payment:** Nếu xảy ra crash hoặc lỗi mạng ngay sau khi vừa ghi checkpoint nhưng trước khi lệnh thanh toán thực sự thành công ở ngân hàng, worker khởi động lại sẽ thấy checkpoint "đã xong" và bỏ qua bước thanh toán. Kết quả là khách hàng không nhận được tiền (`lost refund`), vi phạm nghiêm trọng cam kết tài chính.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (1 min) | page | 27 min |
| provider_outage | page | page (0 min) | page | 4 min |
| deploy_blip | none | page (false page) | none | — |
| slow_leak | ticket | none (bỏ sót) | ticket | 1337 min |
| degraded_evening | page | page (214 min) | page | 291 min |
| night_spike | none | page (false page) | none | — |
| small_bump | none | page (false page) | none | — |

- **Error budget 30 ngày:** Tổng số 25,901,755 requests; cho phép lỗi 129,508.8 requests; thực tế lỗi 55,618 requests. Tỷ lệ sẵn sàng đạt 99.7853% (vượt SLO 99.5%). Ngân sách lỗi **đã tiêu: 42.95%**, **còn lại: 57.05%**. Do budget còn dương (> 0), hệ thống **KHÔNG đóng băng release**.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) KHÔNG nên đánh thức ai lúc 11 giờ trưa?**  
  Vì `small_bump` chỉ kéo dài 20 phút và tiêu tốn một phần cực nhỏ ngân sách lỗi (~0.05% error budget tháng). Sau 20 phút hệ thống tự ổn định trở lại. Nếu dùng ngưỡng ngây thơ 5 phút, trực ban sẽ bị đánh thức oan (alert fatigue), làm giảm sự tập trung vào các sự cố thực sự nguy cấp. Burn-rate đa cửa sổ yêu cầu cả cửa sổ dài (1h) lẫn cửa sổ ngắn (5m) cùng cháy ở tốc độ cao thì mới phát page, do đó lọc sạch 100% false page trên các blip ngắn này.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](/reports/gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):**  
Vào lúc t=120 s, cụm triển khai chính `alpha` bị sụt giảm đột ngột 75% năng lực xử lý (GPU) trong vòng 3 phút do sự cố hạ tầng. Do phiên bản client v1 (`NaiveClient`) sử dụng 3 tầng retry chồng chéo với delay cố định và không có circuit breaker hay retry budget, hệ thống đã tự tạo ra một cơn bão retry với tải lên tới 254 req/s dồn vào `alpha`. Hậu quả là dù nhà cung cấp đã khôi phục năng lực phần cứng tại t=300 s, hệ thống vẫn rơi vào trạng thái sập siêu bền (metastable failure) và mất thêm 155 giây nữa mới có thể tự phục hồi.

**Ảnh hưởng:**  
- 1,285 lượt người dùng thất bại hoàn toàn (tỷ lệ khả dụng rớt xuống còn 45.60%).
- Sự cố kéo dài tổng cộng 335 giây (từ t=120 đến t=455).
- Trong 2,362 lượt người dùng của kịch bản, 1,285 lượt lỗi tương đương tiêu tốn 1,285 / 129,508 ≈ 0.99% error budget tháng chỉ trong vòng chưa đầy 6 phút.

**Timeline** (giây ảo — lấy từ CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực GPU | `calls_ok` lập tức sụt từ 10 call/s xuống 2 call/s; hàng đợi tại `alpha` bắt đầu tích tụ |
| 136 | Bão retry bùng nổ | `attempts_alpha` vọt từ 15 lên 53 req/s; `calls_ok` giảm về 0 |
| 143 | Bão retry đạt đỉnh | `attempts_alpha` chạm đỉnh 254 req/s (vượt gấp 12 lần rate limit 20 rps của alpha); `calls_ok` duy trì bằng 0 |
| 145–299 | Tê liệt hoàn toàn | Tải retry duy trì liên tục ở mức 130–250 req/s; người dùng liên tục bị timeout và deadline |
| 300 | alpha hồi phục 100% phần cứng | Phần cứng khôi phục, nhưng `calls_ok` vẫn là 0 vì hàng đợi nghẹt cứng hàng nghìn request cũ mà client đã timeout bỏ đi |
| 455 | Hệ thống bắt đầu phục hồi | Hàng đợi rác được xả hết sau 155 s (`recovery_s = 155.0 s`); `calls_ok` phục hồi trở lại mức bình thường 4–16 call/s |

**Trigger vs root cause:**  
- **Trigger:** Provider `alpha` bị mất 75% năng lực tính toán GPU trong 3 phút (t=120 đến t=300).
- **Root cause:** Kiến trúc phía client thiếu hoàn toàn cơ chế tự vệ (3 tầng retry nhân số lần gọi lên tới 36 lần, delay cố định gây thundering herd, không có circuit breaker để cắt mạch failover, không có retry budget để kiềm chế khuếch đại tải).
- **Cơ chế metastable failure:** Provider `alpha` xử lý theo hàng đợi FIFO và vẫn tiếp tục xử lý các request tồn đọng dù client đã timeout. Khi client ồ ạt retry, hàng nghìn request vô nghĩa lấp đầy hàng đợi. Khi hạ tầng hồi phục tại t=300, server vẫn kiệt sức xử lý các request rác cũ này trong khi client vẫn tiếp tục retry tải mới, tạo thành vòng lặp tự duy trì nghẽn mạch cho tới t=455.

**Điều gì giúp / điều gì làm tệ hơn:**  
- *Làm tệ hơn:* Client retry delay cố định, retry 3 tầng không giới hạn, không failover sang `alpha-eu` hay `beta`.
- *Giúp ích:* `ResilientClient` với Circuit breaker ngắt mạch ngay khi nhận thấy tỷ lệ lỗi/chậm tăng cao, chuyển tải sang `alpha-eu` và `beta`, kết hợp Retry budget và Full jitter backoff giúp hệ thống vượt qua sự cố với `availability = 1.0` và `recovery_s = 0.0 s`.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient` (Circuit Breaker + Retry Budget + Jitter) trên toàn bộ production | Prevent | Backend/Reliability Lead | Đạt `amplification ≤ 1.1` và `availability ≥ 98%` trên băng G2 trong CI |
| 2 | Cài đặt hệ thống Multi-window Burn-rate Alerting (SRE Workbook) thay thế cho alert threshold 5m | Detect | SRE On-call Lead | Phát hiện đúng 100% sự cố lớn, 0 false page trên tập dữ liệu kiểm thử SLI 30 ngày |
| 3 | Cấu hình tự động Failover Ladder sang `alpha-eu` và `beta` khi provider chính bị suy giảm hiệu năng | Mitigate | Infrastructure Team | `P95 latency ≤ 11.0 s` và `recovery_s ≤ 5 s` khi giả lập mất 75% GPU |

## 7. Một điểm yếu còn lại

**Tình huống:** Khi provider chính `alpha` gặp sự cố nặng và tải dồn phần lớn sang `beta` (như trong G6), chi phí vận hành tăng vọt vì giá mỗi call của `beta` đắt hơn 40% (`cost = 1.4` so với `1.0` của `alpha`). Đồng thời, nếu cả 3 model lớn đều đồng loạt quá tải, client hiện tại vẫn tuần tự chờ qua từng nấc thang fallback, làm tăng P99 latency.  
**Giải pháp khắc phục:** 
1. Áp dụng kỹ thuật **Hedging Requests có điều kiện**: Chỉ phát call dự phòng sang `alpha-eu` nếu call đầu tiên tại `alpha` vượt quá P90 latency (~2.0 s), thay vì đợi hết timeout mới failover.
2. Tối ưu hóa chi phí bằng **Dynamic Traffic Shedding & Downscaling**: Đối với các tác vụ `chat` không cần tool calling, ưu tiên chuyển tải sang `gamma-mini` (chi phí chỉ `0.2`), giữ `beta` làm chốt chặn cuối cùng cho các tác vụ phức tạp.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.592 | 192 | 0 | 2 | 0.955 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ngưỡng 0.80 quá lỏng gây ra 181 câu trả lời sai (`wrong_answers: 181` do `other_question`). Ngưỡng 0.97 quá chặt làm giảm hit ratio xuống 0.499 (< 0.50). Ngưỡng 0.92 đảm bảo `wrong_answers = 0` và `cache_hit_ratio = 0.592`. |
| ttl_s | 30 | Giới hạn số câu trả lời cũ khi có silent update ở t=480 (`policy/shipping`). Tại TTL=300 s, `outdated_answers = 95` (vượt trần 15). Tại TTL=30 s, `outdated_answers = 2` (đạt chuẩn ≤ 15). |
| max_stale_s | 300 | Sự cố provider outage kéo dài 120 s (t=240..360). Cửa sổ stale 300 s cho phép phục vụ 192 câu trả lời stale-if-error, nâng `faq_availability` từ 85.29% lên 96.45%. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng similarity:**  
   - Tại `0.80`: `cache_hit_ratio = 0.610`, nhưng có tới `wrong_answers = 181` (toàn bộ do nhầm lẫn câu hỏi khác: `{'other_question': 181}`), `faq_availability = 0.8593`, `cost = 1.015`. Ví dụ câu hỏi về "thời gian hoàn tiền" bị gán nhầm câu trả lời của "chính sách hoàn tiền".
   - Tại `0.92`: `wrong_answers = 0`, `cache_hit_ratio = 0.592`, `cost = 0.955`, `faq_availability = 0.9645` (tối ưu nhất).
   - Tại `0.97`: `wrong_answers = 0`, nhưng `cache_hit_ratio` tụt xuống `0.499` (trượt chuẩn tối thiểu 0.50), `cost` tăng lên `1.018`.

2. **TTL:**  
   - Khi chạy với `ttl_s = 30`: `outdated_answers = 2`, `hit = 0.592`, `cost = 0.955`.
   - Khi chạy với `ttl_s = 60`: `outdated_answers = 3`, `hit = 0.652`, `cost = 0.915`.
   - Khi chạy với `ttl_s = 300`: `outdated_answers = 95` (vượt xa giới hạn 15).
   - *Giải thích:* Webhook chỉ kích hoạt khi hệ thống CMS chủ động gửi thông báo cập nhật (như sự kiện đổi chính sách ở t=150). Đối với các thay đổi ngầm/không báo trước của đối tác vận chuyển tại t=480 (silent update), không có webhook nào được gửi đến. Do đó, **TTL là cơ chế duy nhất** quy định thời điểm xóa bỏ dữ liệu cũ để ép client tái truy vấn dữ liệu mới.

3. **Stale-if-error:**  
   - Khi đặt `max_stale_s: 0` (tắt stale): Trong 120 s mọi provider sập (t=240–360), client không thể phục vụ câu trả lời cũ (`stale_served = 0`), khiến `faq_availability` rớt xuống `0.8529` (FAIL vì < 0.93).
   - Khi đặt `max_stale_s: 300`: Hệ thống phục vụ 192 câu trả lời cũ nhưng an toàn, đưa `faq_availability` lên `0.9645` (PASS).

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?**  
  Thứ nhất, câu hỏi cá nhân liên quan đến trạng thái dữ liệu động và biến động liên tục theo thời gian thực (đơn hàng đổi trạng thái từ đang giao sang đã giao, hoàn tiền đã duyệt hay chưa). Thứ hai, rủi ro lộ lọt dữ liệu nhạy cảm (PII / security risk) giữa các phiên đăng nhập hoặc lỗi hashing user ID là cực kỳ nghiêm trọng. Thứ ba, tỷ lệ lặp lại của chính một người dùng cho cùng một đơn hàng trong vài phút là rất thấp, không đem lại giá trị hit ratio đáng kể nhưng rủi ro trả lời sai trạng thái giao hàng là rất cao.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**  
  On-call không được chỉ nhìn vào metric tổng thể `availability` của user-facing endpoint. Ta phải thiết lập cảnh báo dựa trên:
  1. **Tỷ lệ phục vụ dữ liệu Stale (`stale_served_ratio` hoặc `stale_served > 0 / min`)**: Khi lượng stale response tăng đột biến, chứng tỏ tầng backend LLM đang không phản hồi được và cache đang phải gánh chịu lỗi.
  2. **Tỷ lệ lỗi trực tiếp của Upstream Provider (Upstream LLM Error Rate & Breaker State)**: Cảnh báo ngay khi circuit breaker của provider chính chuyển sang trạng thái `OPEN` hoặc tỷ lệ lỗi của các cuộc gọi cache miss vượt ngưỡng burn-rate.
