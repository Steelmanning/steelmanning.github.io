# Discord scale with Erlang/OTP

# Q1: Tại sao "1 Guild = 1 Process" và Khi Nào Nó Vỡ

---

## Tại sao chọn mô hình này ngay từ đầu

Để hiểu tại sao Discord chọn "1 guild = 1 process", cần hiểu bài toán gốc:

```
Discord về bản chất là pub/sub:
  - Publisher:  1 user gửi message vào guild
  - Subscribers: tất cả members online trong guild đó
  - Challenge:  subscriber list thay đổi liên tục (join/leave/offline)
```

Mô hình "1 process per guild" là mapping **tự nhiên nhất** với Actor Model:

```elixir
# Thế giới thực:          Actor model:
# 1 Discord server    →   1 GenServer process
# Members online      →   State trong process đó
# Message gửi vào     →   Message vào mailbox
# Fanout tới members  →   send/2 từ process đó ra ngoài
```

Lý do kỹ thuật cụ thể:

**1. State consistency không cần lock**

```elixir
# Trong mô hình khác (shared state):
# Thread A đọc member list
# Thread B xóa member → race condition
# → cần mutex, RWLock, CAS operations

# Trong BEAM actor model:
# Guild process là single-threaded
# Mọi thay đổi state đi qua mailbox theo thứ tự
# → linearizability FREE, không cần lock nào
defmodule Discord.Guild.Process do
  use GenServer

  # Mọi operation serialized tự động qua mailbox
  # Không bao giờ có race condition ở đây
  def handle_cast({:member_join, user_id, pid}, state) do
    new_members = Map.put(state.members, user_id, %{pid: pid, status: :online})
    {:noreply, %{state | members: new_members}}
  end

  def handle_cast({:member_leave, user_id}, state) do
    new_members = Map.delete(state.members, user_id)
    {:noreply, %{state | members: new_members}}
  end

  # Không bao giờ có intermediate state bị expose
  # join và leave không thể interleave
end
```

**2. Fault isolation tự nhiên**

```
Guild A process crash  →  chỉ Guild A bị ảnh hưởng
                       →  Supervisor restart Guild A
                       →  Guild B, C, D... không biết gì
                       →  Users trong Guild B không thấy gì
```

**3. Chi phí rất thấp khi guild còn nhỏ**

```elixir
# Discord năm 2015: guild tối đa ~25 users
# 25 users × send/2 = 25 operations
# Mỗi send/2 ~30μs
# Tổng fanout time: 25 × 30μs = 750μs < 1ms
# → Hoàn toàn ổn, không cần optimize gì
```

---

## Bottleneck tuyến tính — cơ chế vật lý của vấn đề

Đây là phần quan trọng nhất mà ít tài liệu giải thích rõ.

### BEAM process là single-threaded — không phải bug, là thiết kế

```
BEAM Scheduler (1 OS thread per CPU core)
│
├── Core 0: chạy Process A (Guild X)
├── Core 1: chạy Process B (Guild Y)
├── Core 2: chạy Process C (User session 1)
└── Core 3: chạy Process D (User session 2)

Guild X process chỉ chạy trên 1 core tại 1 thời điểm
→ Dù máy có 64 cores, Guild X chỉ dùng được 1 core
→ Tất cả work của Guild X phải đi qua 1 hàng đợi (mailbox)
```

### Reduction — đơn vị scheduling của BEAM

```
BEAM không preempt theo thời gian (như OS thread)
BEAM preempt theo "reductions" — đơn vị công việc

Mỗi function call         = 1 reduction
send/2                    = 1 reduction (nhưng có overhead khác)
Pattern matching          = 1 reduction
Mỗi process được cấp phát = 2000 reductions mỗi lần schedule

Khi process dùng hết 2000 reductions:
→ BEAM scheduler de-schedule process đó
→ Chạy process khác
→ Process cũ chờ turn tiếp theo

Đây là lý do send/2 tốn 30–70μs:
→ send/2 gửi xong → BEAM có thể de-schedule ngay
→ Process chờ turn tiếp mới resume
→ Wall clock time = thời gian chờ re-schedule
```

### Fanout cost tăng tuyến tính — và tại sao đây là vấn đề

```elixir
defmodule Discord.Guild.Process do
  def handle_cast({:new_message, message}, state) do
    # Đây là vòng lặp fanout
    # Với N members online, có N lần send/2
    Enum.each(state.online_members, fn {_user_id, %{pid: pid}} ->
      send(pid, {:guild_message, message})
      # Mỗi send/2: 30–70μs
      # Sau mỗi vài send: BEAM có thể preempt
      # Các messages khác vào guild phải đợi
    end)
    {:noreply, state}
  end
end
```

```
Guild size → Fanout time (worst case):

25 users    →  25 × 70μs  =    1.75ms   ✅ fine
1,000       →  1k × 70μs  =   70ms      ⚠️  borderline
10,000      →  10k × 70μs =  700ms      ❌  users thấy lag
30,000      →  30k × 70μs =   2.1s      💀  unacceptable
1,000,000   →  1M × 70μs  =   70s       ☠️  impossible
```

Vấn đề còn tệ hơn vì **guild process không chỉ làm fanout**:

```elixir
# Trong khi đang fanout 30,000 messages,
# guild process CŨNG phải handle:
handle_cast({:member_join, ...})      # user mới join
handle_cast({:member_leave, ...})     # user leave
handle_cast({:voice_state_update, ...}) # voice channel
handle_call({:get_member_list, ...})  # bot query
handle_info({:presence_update, ...})  # status change

# Tất cả đang queue up trong mailbox
# Guild process đang bận fanout → không thể handle
# → Mailbox ngày càng to → latency tăng → OOM
```

---

## Khi nào mô hình vỡ — timeline thực tế của Discord

---

![guild_process_breaking_points](guild_process_breaking_points.svg)

## Vỡ điểm 1: Guild Registry Stampede — 5 triệu sessions vs 10 processes

Vấn đề cốt lõi là khi session process gọi tới guild registry bị timeout, request vẫn nằm trong queue của guild registry. Process sẽ retry sau backoff, nhưng liên tục pile up requests và rơi vào trạng thái không thể recover. Sessions bắt đầu block trên các requests này cho đến khi timeout, trong khi vẫn nhận messages từ các services khác, khiến message queue phình to và cuối cùng OOM toàn bộ Erlang VM gây cascading outage.

```elixir
# Vấn đề: thundering herd vào 10 registry processes
# 5,000,000 sessions × retry_after_timeout = stampede

# Giải pháp Discord: semaphore tự xây bằng BEAM primitives
defmodule Discord.SemaphoreQueue do
  use GenServer
  # Thay vì circuit breaker (cut off hoàn toàn)
  # Discord dùng semaphore: giới hạn concurrent requests
  # → pressure relief mà không mất requests hoàn toàn

  @max_concurrent 50

  def init(_) do
    {:ok, %{
      available: @max_concurrent,
      waiting: :queue.new()
    }}
  end

  def handle_call({:acquire, timeout}, from, state) do
    if state.available > 0 do
      # Slot available → grant immediately
      {:reply, :ok, %{state | available: state.available - 1}}
    else
      # No slot → queue the caller với deadline
      deadline = System.monotonic_time(:millisecond) + timeout
      waiting  = :queue.in({from, deadline}, state.waiting)
      # Không reply ngay → caller bị block cho đến khi có slot
      {:noreply, %{state | waiting: waiting}}
    end
  end

  def handle_cast(:release, state) do
    # Drain expired waiters trước
    {expired, valid_waiting} = drain_expired(state.waiting)
    Enum.each(expired, fn {from, _} ->
      GenServer.reply(from, {:error, :timeout})
    end)

    case :queue.out(valid_waiting) do
      {{:value, {from, _deadline}}, new_waiting} ->
        GenServer.reply(from, :ok)
        {:noreply, %{state | waiting: new_waiting}}
      {:empty, _} ->
        {:noreply, %{state | available: state.available + 1, waiting: valid_waiting}}
    end
  end
end
```

---

## Vỡ điểm 2: Fanout là O(n) trên single process — Manifold fix

```elixir
# TRƯỚC Manifold: guild process tự fanout
# Với 30,000 members trên 10 nodes:
def fanout_naive(members, message) do
  Enum.each(members, fn {_id, %{pid: pid}} ->
    send(pid, message)
    # Mỗi send tới remote node:
    # → serialize ETF
    # → TCP write
    # → 30–70μs
    # → Guild process bị de-schedule sau vài lần
  end)
  # Tổng: 30,000 × 70μs = 2.1 giây
  # Trong 2.1 giây: guild KHÔNG xử lý được gì khác
end

# SAU Manifold: group by node, delegate
def fanout_manifold(members, message) do
  pids = Enum.map(members, fn {_, %{pid: pid}} -> pid end)

  # Manifold.send/2 — drop-in replacement cho Enum.each + send
  # 1. Group PIDs by remote node: O(n) nhưng local, rất nhanh
  # 2. Gửi 1 message tới Manifold.Partitioner trên mỗi node
  # 3. Partitioner fan out locally trên node đó
  # Guild process chỉ gọi send/2 đúng số_nodes lần
  Manifold.send(pids, message)
  # Với 10 nodes: guild chỉ tốn 10 × 70μs = 700μs thay vì 2.1s
end
```

```
Tại sao Manifold giữ được linearizability?

Không có Manifold:
  Guild gửi: user1(NodeA), user2(NodeA), user3(NodeB)
  → 3 send/2 riêng lẻ
  → Trên NodeA: user1 và user2 nhận theo thứ tự gửi ✅

Với Manifold:
  Guild gửi 1 batch tới NodeA: [user1, user2]
  Manifold.Partitioner trên NodeA:
    hash(user1_pid) → worker_1
    hash(user2_pid) → worker_1  (same worker vì consistent hash)
  worker_1 gửi user1 rồi user2 → thứ tự đảm bảo ✅

  Nếu user1 và user2 hash vào worker khác nhau:
    → 2 workers gửi parallel
    → Nhưng Discord chấp nhận: 2 users khác nhau
      không cần ordering guarantee với nhau
    → Chỉ cần: messages TỚI CÙNG 1 USER theo đúng thứ tự ✅
```

---

## Vỡ điểm 3: Memory — guild heap phình to với 1 triệu members

```elixir
# Mô hình gốc: toàn bộ member list trong process heap
defmodule Discord.Guild.Process do
  defstruct [
    :guild_id,
    # 1,000,000 members × ~500 bytes/member = 500MB
    # Nằm trong process heap
    # → GC phải scan toàn bộ 500MB mỗi lần collect
    # → GC pause = hàng giây
    # → Trong GC pause: guild không xử lý được gì
    members: %{}
  ]
end

# Giải pháp: ETS làm "off-heap" storage
defmodule Discord.Guild.ETSBacked do
  def init(guild_id) do
    # ETS table nằm ngoài process heap
    # Nhiều processes có thể đọc đồng thời
    # GC của guild process không cần scan ETS
    table = :ets.new(
      :"guild_members_#{guild_id}",
      [
        :set,
        :public,            # nhiều process đọc được
        :named_table,
        read_concurrency: true,   # optimize cho nhiều readers
        write_concurrency: false  # guild process là writer duy nhất
      ]
    )

    # Process heap chỉ giữ:
    # - recent changes chưa flush vào ETS (~nhỏ)
    # - metadata của guild
    # → GC nhanh, không pause
    {:ok, %{guild_id: guild_id, ets_table: table, pending_changes: []}}
  end

  def handle_cast({:member_join, user_id, data}, state) do
    # Write vào ETS — O(1), không copy vào process heap
    :ets.insert(state.ets_table, {user_id, data})
    {:noreply, state}
  end

  # Worker process có thể đọc ETS trực tiếp
  # mà không cần gửi message cho guild process
  def spawn_everyone_ping_worker(guild_id, channel_id) do
    table = :"guild_members_#{guild_id}"
    Task.async(fn ->
      # Đọc toàn bộ members từ ETS trong worker process riêng
      # Guild process KHÔNG bị block trong quá trình này
      :ets.foldl(fn {user_id, member_data}, acc ->
        if can_see_channel?(member_data, channel_id) do
          [user_id | acc]
        else
          acc
        end
      end, [], table)
    end)
  end
end
```

---

## Vỡ điểm 4: Single-process ceiling — Relay layer

Relay processes duy trì connections tới sessions thay vì guild, và chịu trách nhiệm fanout với permission checks. Mỗi relay handle tối đa 15,000 connected sessions.

```
TRƯỚC relay:                    SAU relay:

Guild Process                   Guild Process
    │                               │
    │ fanout tới                    │ broadcast tới
    │ 1,000,000 sessions            │ N relay processes
    │                               │
    ├── session_1                   ├── Relay_1 (15k sessions)
    ├── session_2                   │     ├── session_1..15000
    ├── ...                         ├── Relay_2 (15k sessions)
    └── session_1000000             │     ├── session_15001..30000
                                    └── Relay_M
                                          └── session_...

Guild tốn: 1,000,000 sends      Guild tốn: M sends (M = 1M/15k ≈ 67)
           = 70 giây                       = 67 × 70μs ≈ 5ms ✅
```

```elixir
defmodule Discord.Guild.RelayManager do
  # Guild chỉ biết về relay PIDs, không biết về individual sessions
  def broadcast_via_relays(relay_pids, message) do
    # Guild gửi tới M relay processes — M rất nhỏ
    Manifold.send(relay_pids, {:relay_broadcast, message})
    # Mỗi relay tự fanout tới 15k sessions của mình
    # Song song, trên nhiều cores, nhiều nodes
  end
end

defmodule Discord.Relay.Process do
  use GenServer

  # Relay xử lý fanout + permission check
  # Giải phóng guild khỏi công việc nặng nhất
  def handle_info({:relay_broadcast, message}, state) do
    state.sessions
    |> Enum.filter(fn {_uid, session} ->
      # Permission check tại relay — không phải guild
      can_receive_message?(session, message)
    end)
    |> Enum.each(fn {_uid, %{pid: pid}} ->
      send(pid, {:new_message, message})
    end)
    {:noreply, state}
  end
end
```

---

## Tóm tắt — khi nào mô hình vỡ và fix gì

```
Users/guild    Vấn đề                      Fix
──────────────────────────────────────────────────────────────
≤ 1,000        Không có                    Không cần
1k–10k         Hash ring lookup chậm       FastGlobal (0.33μs)
10k–30k        Fanout O(n) trên 1 process  Manifold (group by node)
30k–100k       Registry stampede           Semaphore queue
100k–1M        GC pressure từ huge heap    ETS off-heap storage
               @everyone block guild
1M+            Single process ceiling      Relay layer sharding
               (không thể dùng >1 core)    Passive sessions (90% off)
                                           Worker processes + ETS
```

**Insight lớn nhất**: Discord không bao giờ thay đổi mô hình "1 guild = 1 process" — họ **augment** nó. Guild process vẫn là source of truth, nhưng công việc nặng được delegate ra relay, worker, ETS. Đây là cách OTP supervision tree được thiết kế để scale: không rewrite, mà compose thêm layers.

---
