# Unknown Unknowns — Discord Scale với Erlang/OTP


## Unknown Unknowns — Những gì bạn chưa biết mình chưa biết

Dựa trên research từ Discord engineering blog và các nguồn kỹ thuật, đây là những vấn đề **ẩn sâu** mà phần lớn engineers không nghĩ tới khi build hệ thống WebSocket realtime ở scale Discord.

---

### 🔴 Unknown Unknown #1: send/2 KHÔNG free — nó có thể tốn 30–70μs

Discord phát hiện rằng wall clock time của một lần gọi `send/2` có thể dao động từ 30μs đến 70μs do Erlang de-scheduling calling process. Điều này có nghĩa là trong giờ cao điểm, publish một event từ một guild lớn có thể mất từ 900ms đến 2.1 giây. Không ai nói với bạn điều này trong tài liệu Elixir.

---

### 🔴 Unknown Unknown #2: Process monitor bình thường sẽ giết cluster khi node chết

Khi một node có 500,000 processes chết, BEAM sẽ gửi `{:DOWN, ...}` tới **mọi remote monitor** cùng một lúc — tạo ra một storm message khổng lồ làm flood toàn bộ cluster. ZenMonitor được Discord phát triển để giải quyết vấn đề này: khi một process được monitor bởi số lượng lớn remote processes, việc process đó chết có thể gây flood cả node hosting process đó lẫn các node chứa monitoring processes.

---

### 🔴 Unknown Unknown #3: Consistent hash ring lookup có thể tốn 17.5 giây khi reconnect storm

Khi session server crash và restart, chỉ riêng chi phí lookup trên hash ring đã tốn khoảng 30 giây. Discord giải quyết bằng ETS (7μs/lookup) rồi sau đó dùng FastGlobal (0.3μs/lookup) — nhanh hơn ETS 23 lần bằng cách exploit tính năng shared read-only heap của BEAM VM.

---

### 🔴 Unknown Unknown #4: 90% users trong guild lớn là "passive" — và đây là insight thay đổi toàn bộ kiến trúc

Discord phát hiện khoảng 90% user-guild connections trong các server lớn là passive (users không tương tác). Việc tắt notifications cho passive sessions khiến fanout work rẻ hơn 90%, tương đương tăng maximum community size lên ~3x mà không cần thêm hardware.

---

### 🔴 Unknown Unknown #5: Một BEAM process là single-threaded — guild process là bottleneck tuyến tính

Erlang process hoạt động single-threaded, cách duy nhất để parallelize là shard chúng. Discord giải quyết bằng "relay" system — một layer process nằm giữa guild và session, mỗi relay handle tối đa 15,000 connected sessions và chịu trách nhiệm fanout với permission checks thay cho guild process.

---

### 🔴 Unknown Unknown #6: Mnesia — database built-in của Erlang — không scale được trong failure scenarios

Discord thử dùng Mnesia trong production hai lần — ở cả persistent và in-memory mode. Database nodes thường xuyên bị lag trong failure scenarios, đôi khi không thể catch up được. Cuối cùng họ bỏ Mnesia hoàn toàn và dùng GenServer + ETS.

---

### 🔴 Unknown Unknown #7: Distributed tracing thêm overhead ẩn — ngay cả khi 99% traces bị drop

Khi Discord thêm distributed tracing, các guild bận nhất bắt đầu không theo kịp activity. Profiling cho thấy processes tốn thời gian đáng kể để unpack trace context, ngay cả khi 99%+ operations không được sample. Giải pháp: build filter đọc sampling flag từ encoded string mà không deserialize toàn bộ context.

---

### 🔴 Unknown Unknown #8: Guild migration giữa các nodes có thể stall hàng phút

Khi cần hand off một guild process từ machine này sang machine khác (deployment hoặc maintenance), cần copy state của guild — có thể lên tới nhiều GB data. Việc này stall guild process hàng phút. Discord giải quyết bằng ETS: store members trong ETS, spawn worker process để copy data trong khi guild process vẫn tiếp tục xử lý messages.

---

### 🔴 Unknown Unknown #9: Manifold — fanout naïve tạo O(n) network connections từ 1 node

Manifold phân phối công việc gửi messages tới remote nodes của các PIDs, đảm bảo sending process chỉ gọi `send/2` nhiều nhất bằng số remote nodes liên quan. Manifold làm điều này bằng cách group PIDs theo remote node, gửi tới Manifold.Partitioner trên mỗi node đó, partitioner hash PIDs và gửi tới child workers. Không có Manifold, một guild 30,000 users có thể tạo 30,000 network calls từ 1 process.

---

### 🔴 Unknown Unknown #10: FastGlobal exploit BEAM shared heap — đọc data mà không copy

FastGlobal benchmark: `fastglobal get` đạt 0.33μs/op so với `ets get` 7.64μs/op và `agent get` 12.67μs/op — nhanh hơn ETS 23 lần. Tuy nhiên có trade-off ẩn: compile time chậm khi data lớn, và Erlang phải nói chuyện với mỗi process khi purge module.

---

### 🔴 Unknown Unknown #11: @everyone ping trong guild 1 triệu người là bài toán riêng biệt

Khi có everyone ping, Discord cần biết mọi người trong server có thể xem message đó — việc này có thể mất nhiều giây nếu chạy trong guild process. Discord dùng ETS để share member list, spawn worker process riêng để chạy permission check trong khi guild process tiếp tục xử lý việc khác.

---

### 🔴 Unknown Unknown #12: 5 engineers vận hành toàn bộ chat infrastructure cho 200 triệu MAU

Discord chạy cluster 400-500 Elixir machines. Chat infrastructure team chỉ gồm 5 engineers chịu trách nhiệm cho 20+ Elixir services capable of handling millions of concurrent users và pushing dozens of millions of messages per second. Đây là proof mạnh nhất về leverage của BEAM/OTP.

---

## 15 Câu hỏi Deep-Dive cho series tiếp theo

Sắp xếp từ **foundational → advanced → unknown territory**:

---

**Nhóm 1 — Fanout & Guild Architecture**

> **Q1.** Tại sao Discord chọn mô hình "1 guild = 1 Elixir process" thay vì shard guild ngay từ đầu, và khi nào mô hình đó bắt đầu vỡ? Giải thích cơ chế bottleneck tuyến tính của single BEAM process.

> **Q2.** Manifold giải quyết bài toán O(n) network fanout như thế nào ở mức implementation? Tại sao `:erlang.phash2` consistent hashing trong partitioner lại đảm bảo linearizability?

> **Q3.** Passive session là gì, cơ chế phân loại active/passive diễn ra ở đâu trong pipeline, và tại sao 90% passive lại là số thực tế chứ không phải assumption?

> **Q4.** Relay process layer hoạt động như thế nào — relay giữ state gì, guild giữ state gì, và làm sao hai bên sync khi có member join/leave?

---

**Nhóm 2 — Memory & ETS**

> **Q5.** BEAM process copy-on-send có nghĩa là gì trong thực tế khi guild state có vài GB members? Tại sao ETS giải quyết được vấn đề này mà process message passing không làm được?

> **Q6.** FastGlobal exploit "shared read-only heap" của BEAM — cơ chế này hoạt động ở tầng VM như thế nào, và tại sao `module constant` nhanh hơn ETS 23 lần?

> **Q7.** Guild migration giữa các nodes diễn ra như thế nào step-by-step? Làm sao đảm bảo không mất message trong suốt quá trình handoff khi guild state có thể lên tới nhiều GB?

---

**Nhóm 3 — Fault Tolerance & Monitoring**

> **Q8.** ZenMonitor giải quyết "monitor storm" khi node chết như thế nào? Tại sao `Process.monitor/1` built-in của BEAM lại không scale được ở 500,000 processes per node?

> **Q9.** Khi một guild node chết đột ngột, Discord recover trong ~40 giây — cụ thể những bước gì xảy ra theo thứ tự trong 40 giây đó? Ai detect, ai restart, ai redirect sessions?

> **Q10.** Discord thử Mnesia hai lần và thất bại cả hai. Bài học cụ thể về distributed database trong Erlang ecosystem là gì, và tại sao ETS + GenServer lại là đáp án đúng?

---

**Nhóm 4 — Observability & Tracing**

> **Q11.** Distributed tracing trong actor model khác HTTP-based tracing như thế nào? Discord implement "Transport library" wrap message passing để propagate trace context ra sao mà không break linearizability?

> **Q12.** Dynamic sampling dựa trên fanout size là gì — tại sao message đến 1 người sample 100%, nhưng message đến 10,000 người chỉ sample 0.1%? Làm sao implement điều này mà không tạo overhead cho unsampled paths?

---

**Nhóm 5 — Scale limits & Unknown Territory**

> **Q13.** BEAM scheduler preemption dựa trên "reductions" — đây là cơ chế gì, tại sao nó là lý do send/2 tốn 30–70μs, và làm sao Discord tune scheduler để giảm jitter này?

> **Q14.** Khi guild có 1 triệu+ concurrent users, `@everyone` ping trở thành bài toán riêng biệt. Worker process + ETS giải quyết permission check cho 1 triệu users trong bao lâu, và bottleneck tiếp theo sau ETS là gì?

> **Q15.** Với 32 triệu active guilds, Discord không thể giữ tất cả guild processes trong memory — cơ chế lazy load/unload guild process hoạt động như thế nào, và làm sao tránh thundering herd khi một guild "cold start" đột ngột nhận flood traffic?

---

Gõ **"continue"** để bắt đầu Q1: **Tại sao "1 guild = 1 process" và khi nào nó vỡ**.
