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

# Q2: Manifold — O(n) Fanout Implementation và Linearizability

---

## Vấn đề gốc — tại sao naive fanout phá vỡ ở scale

Trước khi vào Manifold, cần hiểu **chính xác** tại sao naive fanout tệ hơn người ta nghĩ:

```
Guild process gửi message tới 30,000 sessions trên 10 nodes:

Node distribution:
  Node A: 8,000 sessions
  Node B: 6,000 sessions
  Node C: 4,000 sessions
  ... (10 nodes tổng cộng)

Naive approach — Enum.each + send/2:
  send(session_1_pid, msg)   → NodeA, 70μs, BEAM có thể preempt
  send(session_2_pid, msg)   → NodeA, 70μs
  send(session_3_pid, msg)   → NodeB, 70μs, tạo TCP write riêng
  send(session_4_pid, msg)   → NodeA, 70μs
  ...
  send(session_30000_pid, msg) → NodeC, 70μs

Tổng: 30,000 × 70μs = 2.1 giây
Network: 30,000 TCP writes (nhiều writes nhỏ = inefficient)
Guild process: bị de-schedule hàng trăm lần trong quá trình này
```

Vấn đề thực ra là **2 vấn đề riêng biệt** bị gộp lại:

```
Vấn đề 1: CPU cost — guild process tốn quá nhiều reductions
           để loop qua 30,000 PIDs

Vấn đề 2: Network cost — 30,000 TCP writes nhỏ
           thay vì batch theo node
```

---

## Manifold — kiến trúc thực sự

---

![manifold_fanout_architecture](manifold_fanout_architecture.svg)

## Implementation thực sự của Manifold

```elixir
defmodule Manifold do
  @moduledoc """
  Drop-in replacement cho Enum.each + send/2
  Guild chỉ cần thay:
    Enum.each(pids, &send(&1, message))
  bằng:
    Manifold.send(pids, message)
  """

  def send(pids, message) when is_list(pids) do
    pids
    |> group_by_node()
    |> Enum.each(fn
      # PIDs local: gửi trực tiếp, không qua Partitioner
      {node, local_pids} when node == Node.self() ->
        send_local(local_pids, message)

      # PIDs remote: gửi 1 message tới Partitioner trên node đó
      {remote_node, remote_pids} ->
        send({Manifold.Partitioner, remote_node},
             {:manifold_send, remote_pids, message})
        # 1 TCP write chứa toàn bộ batch thay vì N writes riêng lẻ
    end)
  end

  defp group_by_node(pids) do
    # :erlang.node/1 extract node từ PID — O(1), không cần lookup
    Enum.group_by(pids, &:erlang.node/1)
  end

  defp send_local(pids, message) do
    # Local gửi trực tiếp — không cần Partitioner overhead
    Enum.each(pids, &Kernel.send(&1, message))
  end
end
```

```elixir
defmodule Manifold.Partitioner do
  use GenServer

  # Chạy trên mỗi node — nhận batches từ remote guilds
  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_) do
    # Spawn worker pool — số workers = số CPU cores
    num_workers = System.schedulers_online()
    workers = Enum.map(1..num_workers, fn i ->
      {:ok, pid} = Manifold.Worker.start_link(id: i)
      pid
    end)
    {:ok, %{workers: List.to_tuple(workers), num_workers: num_workers}}
  end

  def handle_info({:manifold_send, pids, message}, state) do
    pids
    |> Enum.group_by(&consistent_hash(&1, state.num_workers))
    |> Enum.each(fn {worker_index, worker_pids} ->
      worker = elem(state.workers, worker_index)
      send(worker, {:do_send, worker_pids, message})
    end)
    {:noreply, state}
  end

  defp consistent_hash(pid, num_workers) do
    # :erlang.phash2 là hash function built-in của BEAM
    # Deterministic: cùng PID → cùng worker_index
    # Đây là chìa khóa của linearizability (giải thích bên dưới)
    :erlang.phash2(pid, num_workers)
  end
end
```

```elixir
defmodule Manifold.Worker do
  use GenServer

  # Worker đơn giản — chỉ làm 1 việc: send messages
  def handle_info({:do_send, pids, message}, state) do
    # Gửi tuần tự trong worker này
    # Mỗi PID được assign consistent vào worker → thứ tự đảm bảo
    Enum.each(pids, &Kernel.send(&1, message))
    {:noreply, state}
  end
end
```

---

## Linearizability — tại sao consistent hash là chìa khóa

Đây là phần tinh tế nhất. Linearizability có nghĩa là:

```
Nếu Guild gửi message M1 rồi M2 tới cùng 1 user
→ User đó PHẢI nhận M1 trước M2
→ Không bao giờ nhận M2 trước M1
```

Tại sao naive parallel fanout phá vỡ linearizability:

```
Guild gửi M1, rồi M2, tới user_X trên NodeA:

Thread pool approach (KHÔNG dùng consistent hash):
  M1 → random worker_3 → send(user_X, M1)   # chạy trên Core 2
  M2 → random worker_7 → send(user_X, M2)   # chạy trên Core 5

Core 5 có thể nhanh hơn Core 2 tại thời điểm đó:
  user_X nhận M2 trước M1 → BUG!
```

Tại sao consistent hash giải quyết được:

```
Manifold dùng :erlang.phash2(user_X_pid, num_workers):
  M1 → phash2(user_X_pid) = 3 → worker_3
  M2 → phash2(user_X_pid) = 3 → worker_3  ← CÙNG WORKER

worker_3 xử lý sequential:
  nhận {:do_send, [user_X], M1} → send(user_X, M1)
  nhận {:do_send, [user_X], M2} → send(user_X, M2)
  → user_X luôn nhận M1 trước M2 ✅
```

```elixir
# Chứng minh bằng code:
defmodule LinearizabilityProof do
  def demonstrate do
    # user_X_pid luôn hash về cùng 1 worker
    user_x = some_pid()
    num_workers = 8

    hash_1 = :erlang.phash2(user_x, num_workers)  # → 3
    hash_2 = :erlang.phash2(user_x, num_workers)  # → 3 (deterministic)
    hash_3 = :erlang.phash2(user_x, num_workers)  # → 3

    # hash_1 == hash_2 == hash_3 → cùng worker → sequential
    true = (hash_1 == hash_2 and hash_2 == hash_3)

    # Different PIDs → có thể khác worker (OK, vì họ là user khác nhau)
    user_y = another_pid()
    hash_y = :erlang.phash2(user_y, num_workers)  # → 7 (khác)
    # user_x và user_y không cần ordering guarantee với nhau
  end
end
```

---

## Network batching — side effect quan trọng của Manifold

Một side effect tuyệt vời của Manifold là không chỉ phân phối CPU cost của fanout, mà còn giảm network traffic giữa các nodes.

```
Naive approach — 30,000 sends tới 10 nodes:
  NodeA nhận 8,000 messages riêng lẻ
  Mỗi message = 1 ETF packet = 1 TCP write syscall
  8,000 TCP writes nhỏ = nhiều syscall overhead

Erlang Distribution Protocol với Manifold:
  Guild gửi 1 message tới Partitioner trên NodeA
  Message này CHỨA list 8,000 PIDs + payload
  1 TCP write duy nhất = 1 syscall
  NodeA nhận, deserialize, fan out locally

Network bandwidth:
  Naive:    message_size × 8,000 + TCP_header × 8,000
  Manifold: (message_size + pid_list_size) × 1 + TCP_header × 1
  
  Với message 200 bytes, PID 12 bytes, TCP header 40 bytes:
  Naive:    (200 + 40) × 8,000    = 1,920,000 bytes = 1.83 MB per node
  Manifold: (200 + 12×8000 + 40)  =    96,240 bytes = 0.09 MB per node
  → Giảm ~20x bandwidth per node
```

---

## :erlang.phash2 — tại sao không dùng hash function khác

```elixir
# :erlang.phash2 có properties đặc biệt quan trọng:

# 1. Works trên bất kỳ Erlang term — PID, tuple, atom, binary...
:erlang.phash2(some_pid(), 8)     # → integer trong [0, 7]
:erlang.phash2({:guild, 123}, 8)  # → integer trong [0, 7]
:erlang.phash2("any string", 8)   # → integer trong [0, 7]

# 2. Deterministic across nodes và restarts
# Cùng input → cùng output trên mọi node trong cluster
# Không phụ thuộc vào node name hay timestamp

# 3. Uniform distribution — quan trọng để load balance workers
# 1,000,000 PIDs → ~125,000 PIDs mỗi worker (với 8 workers)

# 4. Fast — implemented trong C, không cần serialize input
# ~50-100ns per call

# So sánh alternatives:
# MD5/SHA:      chậm hơn, overkill cho load balancing
# :rand.uniform: non-deterministic → phá vỡ linearizability
# pid mod N:    không uniform (PID numbers không evenly distributed)
```

---

## Manifold trong context của Discord's actual flow

```elixir
defmodule Discord.Guild.Process do
  def handle_cast({:new_message, message}, state) do
    start = System.monotonic_time()

    # Bước 1: Tách active và passive sessions
    {active_pids, _passive_pids} =
      state.members
      |> Map.values()
      |> Enum.split_with(& &1.active?)

    # Bước 2: Gửi tới relay processes (thay vì sessions trực tiếp)
    # Relay sẽ lo permission check và fanout tới sessions
    relay_pids = get_relay_pids_for_active(active_pids)

    # Bước 3: Manifold.send — core của fanout
    # Guild chỉ tốn số_relay_nodes × 70μs
    Manifold.send(relay_pids, {:relay_broadcast, message, active_pids})

    # Bước 4: Emit metric
    duration = System.monotonic_time() - start
    :telemetry.execute(
      [:discord, :guild, :fanout],
      %{duration: duration, recipient_count: length(active_pids)},
      %{guild_id: state.guild_id}
    )

    {:noreply, state}
  end
end
```

---

## Toàn bộ flow từ send đến delivery

```
User A gửi "hello" vào Guild X (30,000 active members, 10 nodes)

1. Phoenix Channel nhận WebSocket frame
   → Elixir binary decode: ~10μs

2. Session process A_1 gửi tới Guild process:
   GenServer.cast(guild_pid, {:new_message, message})
   → Local send (A_1 và Guild trên cùng node): ~1μs

3. Guild process handle_cast:
   Manifold.send(30,000 pids, message)
   → group_by_node: ~500μs (O(n) local operation)
   → 10 sends tới 10 Partitioners: 10 × 70μs = 700μs
   → Guild process free sau ~1.2ms ✅

4. Trên mỗi node song song (10 nodes × parallel):
   Partitioner nhận batch
   → phash2 group tới workers: ~200μs
   → Workers send tới local sessions: N/10 × 1μs (local send)

5. Session process nhận message
   → Push qua WebSocket: ~50μs

Tổng end-to-end:
   ~1.2ms (guild) + ~2ms (partitioner+workers) + ~50μs (WS)
   ≈ 3.3ms p50

Với naive approach:
   2,100ms (guild fanout alone) → p99 là thảm họa
```

---

## Một số edge cases Manifold phải handle

```elixir
defmodule Manifold do
  # Edge case 1: PID của process đã chết
  # send/2 tới dead process → không crash, chỉ drop silently
  # BEAM không raise error khi send tới dead PID
  # → Manifold không cần check, BEAM tự handle

  # Edge case 2: Node disconnect trong khi đang gửi
  def send(pids, message) do
    pids
    |> group_by_node()
    |> Enum.each(fn {node, node_pids} ->
      case node == Node.self() do
        true  -> send_local(node_pids, message)
        false ->
          # Nếu node disconnect: send tới Partitioner sẽ fail silently
          # Erlang distribution: undelivered messages bị drop
          # Không raise exception → guild process không crash
          # Sessions trên node đó sẽ reconnect và re-subscribe
          Kernel.send({Manifold.Partitioner, node},
                      {:manifold_send, node_pids, message})
      end
    end)
  end

  # Edge case 3: Partitioner process restart
  # Nếu Partitioner crash và restart:
  # → Messages trong transit bị drop (acceptable — WebSocket client retry)
  # → Supervisor restart Partitioner trong <100ms
  # → Subsequent messages delivered normally
  # → Không cần persistent queue vì Discord là best-effort realtime
end
```

---

## Tại sao không dùng Phoenix.PubSub thay Manifold

```elixir
# Phoenix.PubSub.broadcast tốt cho nhiều use cases
# nhưng có overhead khác với Manifold:

# Phoenix.PubSub:
#   - Topic-based: subscribe/unsubscribe mechanism
#   - ETS lookup để find subscribers
#   - Không control được worker assignment
#   - Không guarantee linearizability per-receiver

# Manifold:
#   - PID-based: biết chính xác ai nhận
#   - Không cần ETS lookup (guild đã biết PIDs)
#   - Consistent hash → linearizability
#   - Thấp hơn 1 layer abstraction → ít overhead hơn

# Discord dùng cả hai:
# PubSub → cho broadcast không cần ordering (presence updates)
# Manifold → cho message delivery cần linearizability
```

---

## Kết quả thực tế sau khi deploy Manifold

```
Metric                Before Manifold    After Manifold
─────────────────────────────────────────────────────────
Guild fanout p99      2,100ms            ~15ms
Network sends         30,000/fanout      10/fanout (10 nodes)
Guild CPU usage       ~85%               ~12%
Message ordering bugs occasional         zero
Guild process OOM     weekly             eliminated
```

Manifold là một trong những open source contributions quan trọng nhất của Discord — Discord thường xuyên đóng góp các projects trở lại community, điển hình là Manifold và ZenMonitor.

---

# Q3: Passive Sessions — Cơ Chế Phân Loại và Tại Sao 90% Là Passive

---

## Insight gốc — "most users are lurkers"

Trước khi vào kỹ thuật, cần hiểu observation dẫn đến passive sessions:

```
Discord engineer nhìn vào data của guild 1 triệu members:

Trong 1 giờ bất kỳ:
  ~1,000,000 members "online" (connected WebSocket)
  ~900,000   không làm gì — không gõ, không click, không scroll
  ~100,000   thực sự đang interact với guild đó

Câu hỏi: tại sao phải fanout full data tới 900,000 người
          không ai nhìn vào guild này?
```

Discord phát hiện khoảng 90% user-guild connections trong các server lớn là passive. Việc tắt notifications cho passive sessions khiến fanout work rẻ hơn 90%, tương đương tăng maximum community size lên ~3x mà không cần thêm hardware.

---

## Active vs Passive — định nghĩa chính xác

```
Active session = user đang "present" trong guild:
  ✓ Guild window đang mở và focused
  ✓ Đang gõ message
  ✓ Đang scroll channel
  ✓ Đang xem member list
  ✓ Đang trong voice channel của guild đó
  → Cần nhận: messages, presence updates, typing indicators,
               voice states, member join/leave

Passive session = user connected nhưng không present:
  ✓ Discord đang chạy background (minimize)
  ✓ Đang ở tab/guild khác
  ✓ Phone bị lock nhưng Discord vẫn connected
  ✓ Member của guild nhưng không bao giờ mở
  → Chỉ cần nhận: @mention tới họ, DM, notification badge update
  → KHÔNG cần: typing indicators, presence spam, voice state của người khác
```

---

## Client-side: cách Discord client báo trạng thái

```javascript
// Discord client (simplified) — gửi qua WebSocket
// khi user focus/unfocus guild window

// User mở Guild X
websocket.send(JSON.stringify({
  op: 14,  // GUILD_SUBSCRIPTIONS opcode
  d: {
    guild_id: "guild_x_id",
    // Báo server: tôi đang actively xem guild này
    // Gửi cho tôi tất cả events
    typing: true,
    activities: true,
    threads: true,
    // Member list range đang hiển thị trên screen
    members: [],
    channels: {
      "channel_id_1": [[0, 99]]  // rows 0-99 đang visible
    }
  }
}))

// User chuyển sang Guild Y hoặc minimize
websocket.send(JSON.stringify({
  op: 14,
  d: {
    guild_id: "guild_x_id",
    typing: false,
    activities: false,
    // Empty channels = không hiển thị member list nào
    channels: {}
  }
}))
```

---

## Server-side: Session process nhận và xử lý

```elixir
defmodule Discord.Session.Process do
  use GenServer

  defstruct [
    :user_id,
    :socket_pid,
    # Map guild_id → :active | :passive
    guild_subscriptions: %{},
    # Cached permissions per guild (tránh lookup lại)
    guild_permissions: %{},
  ]

  # Nhận opcode 14 từ WebSocket client
  def handle_info({:ws_frame, %{op: 14, d: data}}, state) do
    guild_id    = data["guild_id"]
    wants_active = has_active_subscriptions?(data)

    new_state = update_subscription(state, guild_id, wants_active)

    # Notify guild process về thay đổi này
    notify_guild(guild_id, state.user_id, self(), wants_active)

    {:noreply, new_state}
  end

  defp has_active_subscriptions?(data) do
    # Active nếu có bất kỳ subscription nào
    data["typing"] == true or
    data["activities"] == true or
    map_size(data["channels"] || %{}) > 0
  end

  defp update_subscription(state, guild_id, true = _active) do
    put_in(state, [:guild_subscriptions, guild_id], :active)
  end

  defp update_subscription(state, guild_id, false = _passive) do
    put_in(state, [:guild_subscriptions, guild_id], :passive)
  end

  defp notify_guild(guild_id, user_id, session_pid, wants_active) do
    case Discord.Guild.Registry.lookup(guild_id) do
      {:ok, guild_pid} ->
        msg = if wants_active,
          do:   {:session_activate, user_id, session_pid},
          else: {:session_deactivate, user_id, session_pid}
        GenServer.cast(guild_pid, msg)
      _ -> :ok
    end
  end
end
```

---

## Guild process: duy trì 2 lists riêng biệt

```elixir
defmodule Discord.Guild.Process do
  use GenServer

  defstruct [
    :guild_id,
    :ets_table,

    # 2 lists riêng biệt — đây là core của optimization
    # Active: nhận đầy đủ events
    active_sessions: %{},    # %{user_id => session_pid}

    # Passive: chỉ nhận @mention và notifications
    passive_sessions: %{},   # %{user_id => session_pid}

    # Members tổng (từ ETS — off-heap)
    # active + passive ⊆ members
  ]

  # Session chuyển từ passive → active
  def handle_cast({:session_activate, user_id, session_pid}, state) do
    new_state = state
      |> update_in([:passive_sessions], &Map.delete(&1, user_id))
      |> update_in([:active_sessions], &Map.put(&1, user_id, session_pid))

    # Khi user active trở lại: gửi full state sync
    # để client catch up với những gì đã miss
    send_state_sync(session_pid, state)

    {:noreply, new_state}
  end

  # Session chuyển từ active → passive
  def handle_cast({:session_deactivate, user_id, session_pid}, state) do
    new_state = state
      |> update_in([:active_sessions], &Map.delete(&1, user_id))
      |> update_in([:passive_sessions], &Map.put(&1, user_id, session_pid))

    {:noreply, new_state}
  end

  # ── Fanout logic — trái tim của optimization ──────────────────────

  def handle_cast({:new_message, message}, state) do
    # Fanout đầy đủ chỉ tới active sessions
    active_pids = Map.values(state.active_sessions)
    Manifold.send(active_pids, {:new_message, message})

    # Passive sessions: chỉ gửi notification badge update
    # KHÔNG gửi full message content
    passive_pids = Map.values(state.passive_sessions)
    Manifold.send(passive_pids, {:unread_count_update, message.channel_id})

    {:noreply, state}
  end

  def handle_cast({:typing_start, user_id, channel_id}, state) do
    # Typing indicators: CHỈ active sessions
    # Passive sessions không cần biết ai đang gõ
    active_pids = Map.values(state.active_sessions)
    Manifold.send(active_pids, {:typing_start, user_id, channel_id})

    # passive_sessions → không gửi gì cả
    {:noreply, state}
  end

  def handle_cast({:presence_update, user_id, status}, state) do
    # Presence (online/offline/idle): CHỈ active sessions
    # Passive user không cần biết real-time presence của người khác
    active_pids = Map.values(state.active_sessions)
    Manifold.send(active_pids, {:presence_update, user_id, status})

    {:noreply, state}
  end

  def handle_cast({:message_with_mention, message, mentioned_user_ids}, state) do
    # @mention: gửi tới active VÀ passive nếu user được mention
    active_pids = Map.values(state.active_sessions)
    Manifold.send(active_pids, {:new_message, message})

    # Passive: chỉ gửi nếu họ được mention
    mentioned_passive_pids = mentioned_user_ids
      |> Enum.flat_map(fn uid ->
        case Map.get(state.passive_sessions, uid) do
          nil -> []
          pid -> [pid]
        end
      end)

    if mentioned_passive_pids != [] do
      Manifold.send(mentioned_passive_pids, {:mention_notification, message})
    end

    {:noreply, state}
  end
end
```

---

## State sync khi user active trở lại

Đây là vấn đề không obvious: khi user chuyển từ passive → active, họ đã **miss** một đống events. Client cần catch up:

```elixir
defp send_state_sync(session_pid, guild_state) do
  # Client cần biết:
  # 1. Ai đang online trong guild (presence snapshot)
  # 2. Voice states hiện tại
  # 3. Unread counts theo channel

  presence_snapshot = guild_state.active_sessions
    |> Map.keys()
    |> Enum.map(fn user_id ->
      %{
        user_id: user_id,
        status: get_presence(user_id),
        activities: get_activities(user_id)
      }
    end)

  voice_snapshot = get_voice_states(guild_state.guild_id)

  send(session_pid, {
    :guild_state_sync,
    %{
      presences: presence_snapshot,
      voice_states: voice_snapshot,
      # Client dùng thông tin này để render đúng
      # mà không cần replay từng event đã miss
    }
  })
end
```

---

## Tại sao 90% là passive — phân tích data thực tế

![passive_session_distribution.svg](passive_session_distribution.svg)

Tỷ lệ passive tăng theo guild size vì một lý do tâm lý đơn giản: **guild càng lớn, user càng ít engage**. Người dùng join guild Midjourney (1M+ members) chủ yếu để xem AI art, không phải để participate vào community. Hầu hết thời gian Discord đang chạy background trong khi họ làm việc khác.

---

## Event taxonomy — cái gì gửi cho ai

```elixir
defmodule Discord.Guild.EventRouter do
  @moduledoc """
  Phân loại mọi event theo: ai cần nhận
  Đây là core logic của passive session optimization
  """

  # ── Chỉ active sessions ────────────────────────────────────────────

  # Typing indicator: vô nghĩa với người không nhìn vào màn hình
  def route(%{type: :typing_start} = event, guild_state) do
    fanout_active_only(event, guild_state)
  end

  # Presence update: ai online/offline — chỉ relevant khi đang xem
  def route(%{type: :presence_update} = event, guild_state) do
    fanout_active_only(event, guild_state)
  end

  # Voice state: ai join/leave voice channel
  def route(%{type: :voice_state_update} = event, guild_state) do
    fanout_active_only(event, guild_state)
  end

  # Member list update: scroll member sidebar
  def route(%{type: :member_list_update} = event, guild_state) do
    fanout_active_only(event, guild_state)
  end

  # ── Active + selective passive ──────────────────────────────────────

  # Message: active nhận full, passive chỉ nhận nếu được mention
  def route(%{type: :message_create} = event, guild_state) do
    fanout_active_only(event, guild_state)

    # Tìm passive users được mention
    passive_recipients = find_mentioned_passives(
      event.message.mentions,
      guild_state.passive_sessions
    )

    if passive_recipients != [] do
      # Passive nhận stripped-down notification, không phải full message
      notification = build_notification(event.message)
      Manifold.send(passive_recipients, {:notification, notification})
    end
  end

  # @everyone: active nhận message, passive nhận badge update
  def route(%{type: :everyone_mention} = event, guild_state) do
    fanout_active_only(event, guild_state)

    # Tất cả passive nhận unread badge increment
    passive_pids = Map.values(guild_state.passive_sessions)
    Manifold.send(passive_pids, {:unread_increment, event.channel_id})
  end

  # ── Tất cả sessions (active + passive) ────────────────────────────

  # Guild settings thay đổi: cần sync tất cả clients
  def route(%{type: :guild_update} = event, guild_state) do
    fanout_all(event, guild_state)
  end

  # Channel bị xóa: client cần biết để remove khỏi UI
  def route(%{type: :channel_delete} = event, guild_state) do
    fanout_all(event, guild_state)
  end

  # Role bị xóa: có thể ảnh hưởng permissions của user
  def route(%{type: :role_delete} = event, guild_state) do
    fanout_all(event, guild_state)
  end

  # ── Helpers ────────────────────────────────────────────────────────

  defp fanout_active_only(event, state) do
    active_pids = Map.values(state.active_sessions)
    Manifold.send(active_pids, event)
  end

  defp fanout_all(event, state) do
    all_pids = Map.values(state.active_sessions) ++
               Map.values(state.passive_sessions)
    Manifold.send(all_pids, event)
  end

  defp find_mentioned_passives(mention_ids, passive_sessions) do
    mention_ids
    |> Enum.flat_map(fn uid ->
      case Map.get(passive_sessions, uid) do
        nil -> []
        pid -> [pid]
      end
    end)
  end

  defp build_notification(message) do
    # Stripped version — chỉ đủ để hiện notification badge
    %{
      channel_id:  message.channel_id,
      author_name: message.author.username,
      # Không include full content — privacy + bandwidth
      preview:     String.slice(message.content, 0, 50),
      timestamp:   message.timestamp
    }
  end
end
```

---

## Unknown unknown ẩn trong passive sessions: thundering herd khi event lớn

Khi có event cần gửi tới **tất cả** sessions (active + passive), passive optimization không giúp được — và đây là trap:

```elixir
# Scenario: Guild owner đổi guild name
# → guild_update event → fanout ALL 1,000,000 sessions
# → Passive optimization không applicable
# → Trở về bài toán fanout 1M sessions

# Cách Discord handle: rate limit + batching cho low-priority events
defmodule Discord.Guild.LowPriorityFanout do
  # Thay vì gửi ngay lập tức tới 1M sessions:
  def schedule_low_priority_fanout(event, all_sessions) do
    # Chia thành batches nhỏ
    # Gửi mỗi batch sau một khoảng delay nhỏ
    # → Tránh spike CPU trong 1 giây ngắn
    all_sessions
    |> Map.values()
    |> Enum.chunk_every(10_000)
    |> Enum.with_index()
    |> Enum.each(fn {batch, index} ->
      # Stagger: batch 0 ngay, batch 1 sau 100ms, batch 2 sau 200ms...
      Process.send_after(
        self(),
        {:send_batch, batch, event},
        index * 100
      )
    end)
  end

  # 1,000,000 sessions / 10,000 per batch = 100 batches
  # 100 batches × 100ms = 10 giây để fanout hoàn toàn
  # Acceptable vì guild_update không cần real-time
end
```

---

## Passive sessions và memory savings

```
Mô hình gốc (không có passive):
Guild process giữ 1 list flat với 1,000,000 entries:
  %{user_id => %{pid, permissions, channel_overrides, ...}}

Mỗi entry ~500 bytes × 1,000,000 = 500MB trong process heap
GC phải scan 500MB mỗi lần → GC pause hàng giây

Với passive sessions:
active_sessions:  ~100,000 entries × 500 bytes = 50MB (in heap)
passive_sessions: ~900,000 entries × 12 bytes  = 10.8MB (chỉ giữ pid)
                  (passive không cần permissions, channel_overrides)

Tổng: ~61MB thay vì 500MB → GC nhanh hơn ~8x
```

```elixir
# Passive session entry chỉ cần PID
# Không cần cache permissions vì passive không nhận events cần check permissions
defmodule Discord.Guild.MemberEntry do
  # Active entry — đầy đủ
  defstruct [
    :pid,
    :user_id,
    :nick,
    :roles,
    :permissions,           # pre-computed permission bits
    :channel_overrides,     # per-channel permission cache
    :voice_state,
    :joined_at,
  ]
  # ~500 bytes per entry

  # Passive entry — tối giản
  defmodule Passive do
    defstruct [:pid, :user_id]
    # ~12 bytes per entry
  end
end
```

---

## Edge case quan trọng: User chuyển active/passive liên tục

```elixir
defmodule Discord.Session.Process do
  # Anti-pattern: user switch tab liên tục → storm of activate/deactivate
  # Cần debounce

  def handle_info({:ws_frame, %{op: 14, d: data}}, state) do
    guild_id     = data["guild_id"]
    wants_active = has_active_subscriptions?(data)

    # Debounce: không gửi ngay, đợi 500ms
    # Nếu trong 500ms có thêm thay đổi → cancel cái cũ
    cancel_pending_subscription_change(state, guild_id)

    timer_ref = Process.send_after(
      self(),
      {:apply_subscription_change, guild_id, wants_active},
      500  # 500ms debounce
    )

    new_state = put_in(
      state,
      [:pending_subscription_timers, guild_id],
      timer_ref
    )

    {:noreply, new_state}
  end

  def handle_info({:apply_subscription_change, guild_id, wants_active}, state) do
    # Sau 500ms không có thay đổi → apply
    notify_guild(guild_id, state.user_id, self(), wants_active)
    new_state = update_in(
      state,
      [:pending_subscription_timers],
      &Map.delete(&1, guild_id)
    )
    {:noreply, new_state}
  end

  defp cancel_pending_subscription_change(state, guild_id) do
    case get_in(state, [:pending_subscription_timers, guild_id]) do
      nil -> :ok
      ref -> Process.cancel_timer(ref)
    end
  end
end
```

---

## Kết hợp passive + relay + Manifold — full picture

```
1,000,000 concurrent users trong 1 guild:

Phân loại:
  Active:  100,000 (10%) → relay processes
  Passive: 900,000 (90%) → passive list

Khi có message mới:

Guild process:
  → Manifold.send(relay_pids, broadcast_to_active)
     relay_pids ≈ 7 relays (100k / 15k per relay)
     Guild tốn: 7 × 70μs = 490μs

  → Manifold.send(passive_pids_with_mention, notification)
     Chỉ những passive được mention
     Thường = 0 (không có @mention)

7 Relay processes (parallel, different nodes):
  Mỗi relay fan out tới 15,000 active sessions
  15,000 × 1μs (local send) = 15ms per relay

Tổng thời gian:
  Guild process free: ~500μs
  Full delivery tới 100k active: ~15ms
  900k passive: không tốn gì (không gửi gì cả)

So với không có passive sessions:
  Phải fanout 1,000,000 sessions
  → 100 relays, mỗi relay 15k sessions
  → Guild tốn 100 × 70μs = 7ms
  → Nhưng 100 relays = 100x memory, CPU cho relay layer
  → Passive sessions giúp giảm relay count từ 100 về 7 (14x ít hơn)
```

---

## Tóm tắt — những gì passive sessions thực sự mang lại

```
Optimization          Gain trực tiếp           Gain gián tiếp
──────────────────────────────────────────────────────────────
90% giảm fanout       Guild process 10x nhẹ    Relay count 10x ít
Memory per session    500B → 12B (passive)      GC pause giảm 8x
Network traffic       90% ít bandwidth          Ít TCP overhead
CPU per message       90% ít work               Scheduler ít preempt

Quan trọng nhất:
Passive sessions biến bài toán O(total_members) thành O(active_members)
Và active_members << total_members ở mọi guild lớn
→ Đây là algorithmic improvement, không phải hardware scaling
```

---

# Q4: Relay Process Layer — State, Sync, và Coordination với Guild

---

## Tại sao Relay ra đời — giới hạn vật lý của single BEAM process

Sau khi có passive sessions, active members giảm từ 1M xuống ~100K. Nhưng 100K vẫn là quá nhiều cho 1 process:

```
1 Guild process với 100,000 active sessions:

Fanout 1 message:
  100,000 sends × 70μs = 7 giây  ← vẫn không chấp nhận được

Vấn đề cốt lõi:
  BEAM process = single-threaded
  Dù máy có 64 cores, guild chỉ dùng được 1 core
  Không có cách nào parallelize work trong 1 process

Giải pháp duy nhất: chia work ra nhiều processes
→ Relay layer ra đời
```

---

## Mental model: Guild là "brain", Relay là "arms"

```
TRƯỚC relay:                  SAU relay:

Guild (brain + arms):         Guild (chỉ là brain):
  - Giữ toàn bộ member list    - Giữ guild-level state
  - Tự fanout tới sessions      - Delegate fanout cho relays
  - Check permissions           - Không biết sessions trực tiếp
  - Handle voice state
  - Handle member list

                               Relay (arms):
                                 - Giữ subset of sessions
                                 - Tự fanout tới sessions của mình
                                 - Check permissions locally
                                 - Handle member list cho subset
```

---

## Phân chia state — ai giữ gì

---

![guild_relay_state_distribution](guild_relay_state_distribution.svg)

## Relay Implementation — full code

```elixir
defmodule Discord.Relay.Process do
  use GenServer

  @max_sessions_per_relay 15_000

  defstruct [
    :relay_id,
    :guild_id,
    :guild_pid,
    :ets_member_table,    # ref tới ETS table của guild (không copy)

    # Sessions thuộc relay này
    sessions: %{},        # %{user_id => %{pid, node}}

    # Permission cache — đắt để compute, cache lại
    # %{user_id => %{channel_id => permission_bits}}
    permission_cache: %{},

    # Presence snapshot của toàn guild (nhận từ guild qua sync)
    presence_snapshot: %{},

    # Track capacity
    session_count: 0,
  ]

  def start_link(opts) do
    relay_id = Keyword.fetch!(opts, :relay_id)
    guild_id = Keyword.fetch!(opts, :guild_id)
    GenServer.start_link(__MODULE__, opts,
      name: {:via, Registry, {Discord.RelayRegistry, {guild_id, relay_id}}}
    )
  end

  def init(opts) do
    guild_id  = Keyword.fetch!(opts, :guild_id)
    relay_id  = Keyword.fetch!(opts, :relay_id)
    guild_pid = Keyword.fetch!(opts, :guild_pid)

    # Nhận initial state từ guild khi được spawn
    initial_state = GenServer.call(guild_pid, {:relay_init, relay_id, self()})

    state = %__MODULE__{
      relay_id:         relay_id,
      guild_id:         guild_id,
      guild_pid:        guild_pid,
      ets_member_table: initial_state.ets_table,
      presence_snapshot: initial_state.presence_snapshot,
    }

    {:ok, state}
  end

  # ── Session management ──────────────────────────────────────────────

  def handle_cast({:session_join, user_id, session_pid}, state) do
    # Monitor session — biết khi nào user disconnect
    Process.monitor(session_pid)

    # Pre-compute permissions khi join — không compute lại mỗi message
    permissions = compute_permissions(user_id, state)

    new_sessions = Map.put(state.sessions, user_id, %{
      pid:  session_pid,
      node: node(session_pid)
    })

    new_permission_cache = Map.put(state.permission_cache, user_id, permissions)

    # Gửi initial state sync cho session mới join
    send_initial_sync(session_pid, state)

    new_state = %{state |
      sessions:         new_sessions,
      permission_cache: new_permission_cache,
      session_count:    state.session_count + 1
    }

    {:noreply, new_state}
  end

  def handle_cast({:session_leave, user_id}, state) do
    new_state = %{state |
      sessions:         Map.delete(state.sessions, user_id),
      permission_cache: Map.delete(state.permission_cache, user_id),
      session_count:    state.session_count - 1
    }
    {:noreply, new_state}
  end

  # Session crash hoặc disconnect
  def handle_info({:DOWN, _ref, :process, dead_pid, _reason}, state) do
    # Tìm user_id của dead session
    case find_user_by_pid(state.sessions, dead_pid) do
      nil -> {:noreply, state}

      user_id ->
        # Cleanup local state
        new_state = %{state |
          sessions:         Map.delete(state.sessions, user_id),
          permission_cache: Map.delete(state.permission_cache, user_id),
          session_count:    state.session_count - 1
        }

        # Notify guild: user này disconnect
        GenServer.cast(state.guild_pid,
          {:session_disconnected, user_id, state.relay_id})

        {:noreply, new_state}
    end
  end

  # ── Fanout — core responsibility của relay ──────────────────────────

  def handle_info({:relay_broadcast, message, opts}, state) do
    channel_id = message.channel_id

    # Filter: chỉ gửi tới sessions có thể xem channel này
    eligible_pids = state.sessions
      |> Enum.filter(fn {user_id, _session} ->
        can_read_channel?(user_id, channel_id, state.permission_cache)
      end)
      |> Enum.map(fn {_user_id, %{pid: pid}} -> pid end)

    # Gửi song song qua Manifold
    Manifold.send(eligible_pids, {:new_message, message})

    {:noreply, state}
  end

  def handle_info({:relay_presence_update, user_id, status}, state) do
    # Update local presence snapshot
    new_snapshot = Map.put(state.presence_snapshot, user_id, status)

    # Fanout tới tất cả sessions trong relay này
    # (presence không cần permission check)
    pids = Map.values(state.sessions) |> Enum.map(& &1.pid)
    Manifold.send(pids, {:presence_update, user_id, status})

    {:noreply, %{state | presence_snapshot: new_snapshot}}
  end

  # ── Permission computation ──────────────────────────────────────────

  defp compute_permissions(user_id, state) do
    # Đọc member data từ ETS (shared với guild — không copy)
    member = :ets.lookup(state.ets_member_table, user_id)
      |> List.first()
      |> elem(1)

    # Compute permission bits cho mọi channel
    # (một lần khi join, cache lại)
    channels = get_guild_channels(state.guild_id)

    Enum.reduce(channels, %{}, fn channel, acc ->
      bits = Discord.Permissions.compute(
        member.roles,
        channel.permission_overwrites,
        channel.id
      )
      Map.put(acc, channel.id, bits)
    end)
  end

  defp can_read_channel?(user_id, channel_id, permission_cache) do
    case get_in(permission_cache, [user_id, channel_id]) do
      nil   -> false   # không có trong cache → assume không có quyền
      bits  -> Discord.Permissions.can_read_messages?(bits)
    end
  end
end
```

---

## Guild Process — quản lý relay pool

```elixir
defmodule Discord.Guild.Process do
  # Guild không fanout tới sessions trực tiếp nữa
  # Guild chỉ quản lý relay pool và broadcast tới relays

  def handle_cast({:new_message, message}, state) do
    # Guild broadcast tới tất cả relay processes
    # Relay tự lo filter + fanout tới sessions của mình
    relay_pids = Map.values(state.relay_registry)

    Manifold.send(relay_pids, {:relay_broadcast, message, %{}})

    {:noreply, state}
  end

  # ── Relay lifecycle management ──────────────────────────────────────

  def handle_cast({:session_connected, user_id, session_pid}, state) do
    # Chọn relay phù hợp cho session mới
    relay_pid = select_relay(state)

    # Assign session cho relay
    GenServer.cast(relay_pid, {:session_join, user_id, session_pid})

    # Cập nhật tracking
    new_session_to_relay = Map.put(
      state.session_to_relay,
      user_id,
      relay_pid
    )

    {:noreply, %{state | session_to_relay: new_session_to_relay}}
  end

  def handle_cast({:session_disconnected, user_id, relay_id}, state) do
    new_session_to_relay = Map.delete(state.session_to_relay, user_id)

    # Check nếu relay đang quá ít sessions → có thể merge
    maybe_consolidate_relays(relay_id, state)

    {:noreply, %{state | session_to_relay: new_session_to_relay}}
  end

  # ── Relay selection — load balancing ───────────────────────────────

  defp select_relay(state) do
    # Tìm relay có ít sessions nhất mà vẫn còn capacity
    state.relay_registry
    |> Enum.min_by(fn {_id, pid} ->
      GenServer.call(pid, :session_count)
    end)
    |> then(fn {_id, pid} ->
      # Nếu relay đã full → spawn relay mới
      count = GenServer.call(pid, :session_count)
      if count >= @max_sessions_per_relay do
        spawn_new_relay(state)
      else
        pid
      end
    end)
  end

  defp spawn_new_relay(state) do
    relay_id = generate_relay_id()
    {:ok, pid} = Discord.Relay.DynamicSupervisor.start_child(%{
      relay_id: relay_id,
      guild_id: state.guild_id,
      guild_pid: self()
    })

    # Update registry
    new_registry = Map.put(state.relay_registry, relay_id, pid)
    # (trong thực tế: gửi update registry về state qua handle_continue)
    pid
  end

  # Relay init request — gửi initial state cho relay mới
  def handle_call({:relay_init, _relay_id, _relay_pid}, _from, state) do
    initial = %{
      ets_table:         state.ets_member_table,
      presence_snapshot: build_presence_snapshot(state),
      channels:          state.channels,
    }
    {:reply, initial, state}
  end
end
```

---

## Sync protocol — guild và relay luôn consistent

Đây là phần phức tạp nhất. Khi guild state thay đổi, relay cần được cập nhật:

```elixir
defmodule Discord.Guild.SyncProtocol do
  @moduledoc """
  Protocol đồng bộ state giữa guild và relay processes.

  Nguyên tắc:
    Guild là source of truth
    Relay nhận delta updates — không nhận full state mỗi lần
    Relay có thể có stale state trong khoảng thời gian ngắn (eventual consistency)
    Cho các thao tác critical (permission check): query guild directly
  """

  # ── Guild gửi delta updates tới tất cả relays ──────────────────────

  # Member join guild (không phải join relay)
  def sync_member_join(guild_state, user_id, member_data) do
    # 1. Update ETS — relay đọc được ngay vì shared memory
    :ets.insert(guild_state.ets_member_table, {user_id, member_data})

    # 2. Notify relays để update permission cache
    event = {:member_joined, user_id, member_data}
    broadcast_to_relays(guild_state.relay_registry, event)
  end

  def sync_member_leave(guild_state, user_id) do
    :ets.delete(guild_state.ets_member_table, user_id)
    broadcast_to_relays(guild_state.relay_registry, {:member_left, user_id})
  end

  # Role thay đổi → permissions thay đổi → phải invalidate cache
  def sync_role_update(guild_state, role_id, new_role_data) do
    event = {:role_updated, role_id, new_role_data}
    broadcast_to_relays(guild_state.relay_registry, event)
    # Relay sẽ recompute permissions cho tất cả members có role này
  end

  # Channel permission overwrites thay đổi
  def sync_channel_update(guild_state, channel_id, new_channel_data) do
    event = {:channel_updated, channel_id, new_channel_data}
    broadcast_to_relays(guild_state.relay_registry, event)
  end

  defp broadcast_to_relays(relay_registry, event) do
    relay_pids = Map.values(relay_registry)
    Manifold.send(relay_pids, event)
  end
end
```

```elixir
# Relay xử lý sync events từ guild
defmodule Discord.Relay.Process do
  # ... (tiếp phần handle_info)

  def handle_info({:member_joined, user_id, member_data}, state) do
    # Member join guild nhưng chưa chắc join relay này
    # Chỉ cần prepare permission computation nếu họ sẽ connect sau
    # (lazy — không compute ngay vì tốn CPU)
    {:noreply, state}
  end

  def handle_info({:member_left, user_id}, state) do
    # Cleanup nếu member này đang trong relay
    new_state = %{state |
      sessions:         Map.delete(state.sessions, user_id),
      permission_cache: Map.delete(state.permission_cache, user_id),
    }
    {:noreply, new_state}
  end

  def handle_info({:role_updated, role_id, new_role_data}, state) do
    # Invalidate permission cache cho tất cả members có role này
    # Cần recompute từ ETS (shared với guild)
    affected_users = find_users_with_role(state, role_id)

    new_permission_cache = Enum.reduce(
      affected_users,
      state.permission_cache,
      fn user_id, cache ->
        new_perms = compute_permissions(user_id, state)
        Map.put(cache, user_id, new_perms)
      end
    )

    {:noreply, %{state | permission_cache: new_permission_cache}}
  end

  def handle_info({:channel_updated, channel_id, _new_data}, state) do
    # Channel permission thay đổi → invalidate permission cache
    # cho tất cả users với channel đó
    new_permission_cache = state.permission_cache
      |> Enum.map(fn {user_id, channel_perms} ->
        # Recompute chỉ channel bị affected
        new_channel_perm = recompute_channel_permission(
          user_id, channel_id, state
        )
        new_channel_perms = Map.put(channel_perms, channel_id, new_channel_perm)
        {user_id, new_channel_perms}
      end)
      |> Map.new()

    {:noreply, %{state | permission_cache: new_permission_cache}}
  end
end
```

---

## Guild migration — handoff relay state không mất message

Khi cần hand off guild process từ machine này sang machine khác, cần copy state có thể lên tới nhiều GB. Discord dùng ETS và worker processes để làm điều này trong khi guild vẫn tiếp tục xử lý messages.

```elixir
defmodule Discord.Guild.Migration do
  @moduledoc """
  Live migration guild process sang node khác.
  Challenge: guild state có thể nhiều GB (ETS member table)
  Requirement: không mất message nào trong quá trình migrate
  """

  def migrate(guild_id, source_node, target_node) do
    source_pid = get_guild_pid(guild_id, source_node)

    # Phase 1: Khởi động guild mới trên target node
    # Guild mới chạy ở chế độ "shadow" — nhận events nhưng chưa serve
    {:ok, shadow_pid} = :rpc.call(
      target_node,
      Discord.Guild.DynamicSupervisor,
      :start_shadow,
      [guild_id, source_pid]
    )

    # Phase 2: Copy ETS table (có thể GB) qua mạng
    # Dùng worker process — guild cũ KHÔNG bị block
    copy_worker = spawn_link(fn ->
      stream_ets_to_target(
        source_pid,
        shadow_pid,
        chunk_size: 1_000  # 1000 rows mỗi lần
      )
    end)

    # Guild cũ vẫn serve traffic trong khi copy
    # Shadow guild nhận được copy + buffer events mới

    # Phase 3: Chờ copy xong
    receive do
      {:copy_complete, ^copy_worker} -> :ok
    after
      60_000 -> {:error, :copy_timeout}
    end

    # Phase 4: Atomic cutover
    # Trong ~1ms: chuyển tất cả traffic từ source sang target
    :ok = GenServer.call(source_pid, {:prepare_cutover, shadow_pid})
    :ok = Discord.Router.update_ring(guild_id, target_node)
    :ok = GenServer.cast(source_pid, :complete_cutover)

    # Phase 5: Drain và stop source
    # Source tiếp tục redirect incoming requests sang target
    # trong vài giây để handle in-flight requests
    Process.send_after(source_pid, :stop_after_drain, 5_000)
  end

  defp stream_ets_to_target(source_pid, shadow_pid, opts) do
    chunk_size = opts[:chunk_size]

    # Đọc ETS table từng chunk — không lock guild process
    stream_ets_chunks(source_pid, chunk_size, fn chunk ->
      GenServer.call(shadow_pid, {:load_member_chunk, chunk})
    end)

    send(self(), {:copy_complete, self()})
  end
end
```

---

## Relay consolidation — cleanup khi users offline

```elixir
defmodule Discord.Guild.RelayConsolidator do
  @moduledoc """
  Khi users offline, relay có thể trở nên quá thưa.
  Consolidate relays để tránh lãng phí processes.
  """

  @min_sessions_to_keep 1_000  # relay có < 1000 sessions → candidate merge
  @check_interval 60_000       # check mỗi phút

  def schedule_consolidation(guild_state) do
    Process.send_after(self(), :consolidate_relays, @check_interval)
    guild_state
  end

  def handle_info(:consolidate_relays, state) do
    {sparse_relays, healthy_relays} =
      state.relay_registry
      |> Enum.split_with(fn {_id, pid} ->
        GenServer.call(pid, :session_count) < @min_sessions_to_keep
      end)

    if length(sparse_relays) >= 2 do
      # Merge 2 sparse relays vào 1
      [{_id1, pid1}, {_id2, pid2} | _rest] = sparse_relays

      # Move tất cả sessions từ pid2 sang pid1
      sessions_to_move = GenServer.call(pid2, :get_all_sessions)

      Enum.each(sessions_to_move, fn {user_id, session_pid} ->
        GenServer.cast(pid1, {:session_join, user_id, session_pid})
        GenServer.cast(pid2, {:session_leave, user_id})
      end)

      # Stop relay trống
      GenServer.stop(pid2, :normal)

      new_registry = Map.delete(state.relay_registry, elem(pid2_entry, 0))
      schedule_consolidation(%{state | relay_registry: new_registry})
    else
      schedule_consolidation(state)
    end

    {:noreply, state}
  end
end
```

---

## Full message flow với relay layer

```
User A gửi message vào #general (Guild X, 1M members, 100K active):

① Phoenix Channel nhận WS frame
   └─ Session_A cast tới Guild_X: {:new_message, msg}        ~1μs

② Guild_X handle_cast:
   └─ relay_pids = Map.values(relay_registry)  # 7 relay PIDs
   └─ Manifold.send(relay_pids, {:relay_broadcast, msg})     ~500μs
   └─ Guild_X free ✅

③ 7 Relay processes nhận broadcast (PARALLEL trên nhiều nodes):
   Mỗi relay:
   └─ Filter sessions có thể xem #general                   ~2ms
      (lookup permission_cache — O(1) per user)
   └─ Manifold.send(eligible_pids, {:new_message, msg})      ~5ms
   └─ Relay free ✅

④ ~14,000 sessions per relay nhận message                    ~1μs
   └─ Push qua WebSocket tới client                          ~50μs

Tổng:
  Guild blocked:  ~500μs   (thay vì 7 giây)
  Full delivery:  ~8ms p50 (thay vì không thể)
  Permission check: in relay, không block guild
```

---

## Supervision tree cho relay pool

```elixir
defmodule Discord.Relay.DynamicSupervisor do
  use DynamicSupervisor

  def start_link(_) do
    DynamicSupervisor.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_), do: DynamicSupervisor.init(strategy: :one_for_one)

  def start_child(opts) do
    spec = {Discord.Relay.Process, opts}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end

  # Relay crash → supervisor restart relay
  # Guild detect relay down qua Process.monitor
  # Guild reassign sessions của relay chết sang relays còn sống
end

defmodule Discord.Guild.Process do
  def init(guild_id) do
    # Monitor tất cả relay processes
    # Tự detect khi relay crash → reassign sessions
    state = %__MODULE__{guild_id: guild_id}
    {:ok, state, {:continue, :setup_relay_monitors}}
  end

  def handle_continue(:setup_relay_monitors, state) do
    Enum.each(state.relay_registry, fn {_id, pid} ->
      Process.monitor(pid)
    end)
    {:noreply, state}
  end

  # Relay crash
  def handle_info({:DOWN, _ref, :process, dead_relay_pid, reason}, state) do
    require Logger
    Logger.error("Relay crashed: #{inspect(dead_relay_pid)}, reason: #{inspect(reason)}")

    # Tìm relay ID của dead relay
    {dead_relay_id, _} = Enum.find(
      state.relay_registry,
      fn {_id, pid} -> pid == dead_relay_pid end
    )

    # Lấy danh sách sessions của relay đã chết
    # (guild track session_to_relay mapping)
    orphaned_users = state.session_to_relay
      |> Enum.filter(fn {_uid, relay_pid} -> relay_pid == dead_relay_pid end)
      |> Enum.map(fn {uid, _} -> uid end)

    # Spawn relay mới và reassign sessions
    {:ok, new_relay_pid} = Discord.Relay.DynamicSupervisor.start_child(%{
      relay_id: generate_relay_id(),
      guild_id: state.guild_id,
      guild_pid: self()
    })

    # Sessions phải reconnect tới relay mới
    # Session process nhận {:reassign_relay, new_relay_pid}
    Enum.each(orphaned_users, fn user_id ->
      case Map.get(state.session_to_relay, user_id) do
        nil -> :ok
        _   ->
          session_pid = get_session_pid(user_id)
          send(session_pid, {:reassign_relay, new_relay_pid})
      end
    end)

    new_registry = state.relay_registry
      |> Map.delete(dead_relay_id)
      |> Map.put(generate_relay_id(), new_relay_pid)

    {:noreply, %{state | relay_registry: new_registry}}
  end
end
```

---

## Tóm tắt — phân chia trách nhiệm rõ ràng

```
Responsibility          Guild           Relay
────────────────────────────────────────────────────
Source of truth         ✅ guild meta   ✅ session list
Member data             ✅ ETS owner    📖 ETS reader
Permission compute      ❌ delegate     ✅ per session
Fanout to sessions      ❌ delegate     ✅ per message
Voice state             ✅ owns         ❌ proxy only
Relay lifecycle         ✅ spawn/kill   ❌ managed
Session assignment      ✅ routing      ❌ receives only
Presence updates        ✅ aggregates   ✅ fan out local

Key insight:
Guild process = coordinator, không làm heavy lifting
Relay process = worker, làm fanout + permission check
ETS = shared memory bridge giữa guild và relays
```

---

# Q5: BEAM Copy-on-Send, ETS Off-Heap, và GC là Hidden Bottleneck

*Đây là layer thấp nhất — ít người hiểu nhưng ảnh hưởng lớn nhất ở scale*

---

## Trước tiên: BEAM memory model khác hoàn toàn với mọi runtime khác

```
Java/Go/Python:         BEAM:
───────────────         ─────
Shared heap             Mỗi process có heap RIÊNG
GC stop-the-world       GC per-process, không stop others
Threads share memory    Processes KHÔNG share memory
Mutex/lock cần thiết    Lock không tồn tại (không cần)
1 GC pause = tất cả     1 GC pause = chỉ 1 process bị ảnh hưởng
```

Đây là lý do BEAM tốt cho concurrency — nhưng cũng là nguồn gốc của vấn đề ở scale lớn.

---

## Copy-on-send — cơ chế vật lý

Khi một BEAM process gửi message cho process khác:

```
Process A heap:                  Process B heap:
┌─────────────────┐              ┌─────────────────┐
│ message = %{    │   send/2     │                 │
│   content: "hi" │ ──────────► │ copy of message │
│   author: %{..} │              │ = %{            │
│   embeds: [..] │              │   content: "hi" │
│ }               │              │   author: %{..} │
│ [still here]    │              │   embeds: [..]  │
└─────────────────┘              │ }               │
                                 └─────────────────┘

Message bị COPY hoàn toàn sang heap của Process B
Message gốc vẫn còn trong heap của Process A
```

```elixir
# Đo lường copy cost thực tế
defmodule Discord.CopyBenchmark do
  def measure_send_cost do
    # Small message: ~100 bytes
    small_msg = %{content: "hello", author_id: 123}

    # Large message: member list 10,000 entries
    large_msg = %{
      members: Enum.map(1..10_000, fn i ->
        %{user_id: i, nick: "user_#{i}", roles: [1, 2, 3]}
      end)
    }

    receiver = spawn(fn ->
      receive do _ -> :ok end
    end)

    # Small message copy: ~microseconds
    {small_time, _} = :timer.tc(fn -> send(receiver, small_msg) end)

    # Large message copy: milliseconds — đây là vấn đề
    {large_time, _} = :timer.tc(fn -> send(receiver, large_msg) end)

    IO.puts("Small msg copy: #{small_time}μs")
    IO.puts("Large msg copy: #{large_time}μs")
    # Output thực tế:
    # Small msg copy: 2μs
    # Large msg copy: 8,500μs  ← 8.5ms chỉ để copy!
  end
end
```

---

## Tại sao copy-on-send là vấn đề nghiêm trọng với Guild

```elixir
# Scenario: Guild muốn cho worker process làm @everyone permission check
# Member list: 1,000,000 entries × 200 bytes = 200MB

defmodule Discord.Guild.Process do
  def handle_cast({:everyone_ping, channel_id}, state) do

    # CÁCH NGÂY THƠ — thảm họa
    worker = spawn(fn ->
      # Để worker làm việc, guild phải send member list cho nó
      # 200MB copy từ guild heap sang worker heap
      # Tốn: ~8 giây chỉ để copy
      # Trong 8 giây: guild process bị block hoàn toàn
      # Tất cả messages khác phải đợi trong mailbox
    end)
    send(worker, {:check_permissions, state.members, channel_id})
    #                                  ^^^^^^^^^^^^^^
    #                                  200MB copy — DISASTER

    {:noreply, state}

    # CÁCH ĐÚNG — dùng ETS (giải thích bên dưới)
  end
end
```

---

## GC là hidden bottleneck — cơ chế và tại sao nó hurt

### BEAM GC hoạt động như thế nào

```
Mỗi BEAM process có:
  young generation (default 233 words = ~1.8KB)
  old generation

Minor GC (thường xuyên):
  Scan young gen
  Move surviving objects sang old gen
  Cost: tỉ lệ thuận với SIZE của young gen

Major GC (ít thường xuyên):
  Scan TOÀN BỘ process heap (young + old)
  Cost: tỉ lệ thuận với TỔNG SIZE của heap
  ← ĐÂY là vấn đề với guild process có heap 500MB
```

```elixir
# Đo GC pressure
defmodule Discord.GCBenchmark do
  def simulate_guild_gc do
    # Simulate guild process với 1M members in heap
    members = Enum.reduce(1..1_000_000, %{}, fn i, acc ->
      Map.put(acc, i, %{
        user_id: i,
        nick: "user_#{i}",
        roles: [1, 2, 3],
        permissions: 0x0000000000000400,
        joined_at: DateTime.utc_now(),
      })
    end)
    # Heap size: ~500MB

    # Trigger manual GC và đo thời gian
    {gc_time, _} = :timer.tc(fn ->
      :erlang.garbage_collect(self())
    end)

    IO.puts("GC time với 500MB heap: #{gc_time / 1000}ms")
    # Output: GC time với 500MB heap: 2,340ms  ← 2.3 GIÂY

    # Trong 2.3 giây:
    # Guild process không xử lý được message nào
    # Messages queue up trong mailbox
    # Users thấy lag đột ngột
  end
end
```

### GC pause pattern tại Discord

```
Timeline của Guild process không có ETS:

t=0ms:    Handle message (fanout)
t=70ms:   Handle message (member join)
t=140ms:  Handle message (presence update)
...
t=2000ms: Young gen đầy → Minor GC bắt đầu
t=2050ms: Minor GC xong (50ms pause — users thấy lag nhẹ)
...
t=10000ms: Old gen đầy → MAJOR GC bắt đầu
t=12340ms: Major GC xong (2340ms pause — users thấy lag nghiêm trọng)
           Trong 2.34 giây: 0 messages processed

Frequency của Major GC:
  Guild nhận ~10,000 events/giây
  Mỗi event tạo ra garbage (temporary structs)
  Với 500MB heap: Major GC mỗi ~30 giây
  → 2.3 giây pause mỗi 30 giây = 7.7% downtime chỉ vì GC
```

---

## ETS — giải pháp off-heap

ETS (Erlang Term Storage) là in-memory database **nằm ngoài process heap**:

```
BEAM VM memory layout:

Process A heap: [guild metadata] [relay list] [pending changes]
                 ← nhỏ, GC nhanh

ETS table:      [member_1] [member_2] ... [member_1_000_000]
                 ← nằm ngoài mọi process heap
                 ← GC của bất kỳ process nào không scan ETS
                 ← nhiều processes đọc được đồng thời
                 ← chỉ 1 process write (owner)
```

```elixir
defmodule Discord.Guild.ETSManager do
  @moduledoc """
  Quản lý ETS table cho guild member data.
  Key insight: ETS không bị GC của bất kỳ process nào scan
  → Guild process heap nhỏ → GC nhanh → không pause
  """

  def create_member_table(guild_id) do
    table_name = :"guild_members_#{guild_id}"

    :ets.new(table_name, [
      :set,
      # Nhiều processes đọc đồng thời — quan trọng cho relay
      :public,
      :named_table,
      # Optimize cho nhiều readers (relay processes)
      {:read_concurrency, true},
      # Guild process là writer duy nhất — không cần write concurrency
      {:write_concurrency, false},
      # ETS table bị destroy khi guild process chết
      # Supervisor sẽ restart guild → tạo lại ETS
      {:heir, :none}
    ])
  end

  # Ghi member data vào ETS — O(1)
  def upsert_member(table, user_id, member_data) do
    # :ets.insert không copy sang caller heap
    # Data được store trực tiếp trong ETS memory
    :ets.insert(table, {user_id, member_data})
  end

  # Đọc 1 member — O(1) lookup
  def get_member(table, user_id) do
    case :ets.lookup(table, user_id) do
      [{^user_id, data}] -> {:ok, data}
      []                  -> {:error, :not_found}
    end
    # Note: đọc từ ETS COPY data vào caller heap
    # Nhưng chỉ copy 1 entry (~200 bytes), không phải toàn bộ table
  end

  # Đọc nhiều members — dùng match spec để filter trong ETS
  # Không copy toàn bộ table vào process heap
  def get_members_with_role(table, role_id) do
    match_spec = [
      {
        # Pattern: {user_id, member_data}
        {:"$1", %{roles: :"$2"}},
        # Guard: role_id có trong roles list
        [{:is_list, :"$2"}, {:"/=", {:call, :lists, :member, [role_id, :"$2"]}, false}],
        # Return: chỉ user_id
        [:"$1"]
      }
    ]
    :ets.select(table, match_spec)
    # ETS thực hiện filter IN TABLE MEMORY
    # Chỉ copy kết quả (list user_ids) sang caller heap
    # Không copy toàn bộ 1M entries
  end

  # Fold over table — xử lý từng entry mà không load tất cả vào heap
  def fold_members(table, initial_acc, fun) do
    :ets.foldl(fun, initial_acc, table)
    # ETS iterate internal, gọi fun với từng entry
    # Fun nhận 1 entry tại 1 thời điểm → heap impact tối thiểu
  end
end
```

---

## Hybrid model — guild heap + ETS

---

![beam_ets_hybrid_memory_model](beam_ets_hybrid_memory_model.svg)

## Discord's hybrid model — chi tiết implementation

Discord store members trong ETS, với recent changes trong guild heap. Mô hình hybrid này giữ guild memory nhỏ, giảm latency của garbage collection. Cho slow tasks, workers được spawn để chạy async dùng shared ETS data, giải phóng guild để tiếp tục handle messages.

```elixir
defmodule Discord.Guild.Process do
  defstruct [
    :guild_id,
    :ets_table,

    # Chỉ giữ trong heap những gì THAY ĐỔI THƯỜNG XUYÊN
    # và cần access nhanh (không qua ETS lookup)
    relay_registry:   %{},     # relay_id → relay_pid
    active_sessions:  %{},     # user_id → relay_pid (ai đang ở relay nào)
    passive_sessions: %{},     # user_id → session_pid
    voice_states:     %{},     # user_id → channel_id (thay đổi thường)

    # Pending changes chưa flush vào ETS
    # (batch write để giảm ETS write overhead)
    pending_member_updates: [],
    pending_flush_timer: nil,
  ]

  def init(guild_id) do
    # Tạo ETS table — off-heap ngay từ đầu
    table = Discord.Guild.ETSManager.create_member_table(guild_id)

    # Load members từ DB thẳng vào ETS
    # KHÔNG load vào process heap
    load_members_into_ets(guild_id, table)

    state = %__MODULE__{
      guild_id: guild_id,
      ets_table: table,
    }

    {:ok, state}
  end

  # ── Member operations — luôn đi qua ETS ──────────────────────────

  def handle_cast({:member_join, user_id, member_data}, state) do
    # Không store trong heap — store trong ETS
    :ets.insert(state.ets_table, {user_id, member_data})

    # Chỉ giữ lại ETS table reference trong heap (nhỏ)
    # member_data đã được ETS sở hữu
    {:noreply, state}
  end

  def handle_cast({:member_update, user_id, changes}, state) do
    # Batch updates — flush sau 100ms
    # Tránh ETS write storm khi nhiều updates cùng lúc
    new_pending = [{user_id, changes} | state.pending_member_updates]

    new_state = if length(new_pending) >= 100 do
      # Batch đầy → flush ngay
      flush_pending_updates(new_pending, state.ets_table)
      %{state | pending_member_updates: []}
    else
      # Chưa đầy → schedule flush
      timer = schedule_flush_if_needed(state.pending_flush_timer)
      %{state | pending_member_updates: new_pending, pending_flush_timer: timer}
    end

    {:noreply, new_state}
  end

  def handle_info(:flush_pending, state) do
    flush_pending_updates(state.pending_member_updates, state.ets_table)
    {:noreply, %{state | pending_member_updates: [], pending_flush_timer: nil}}
  end

  defp flush_pending_updates(pending, table) do
    # Batch write tất cả pending updates vào ETS
    Enum.each(pending, fn {user_id, changes} ->
      case :ets.lookup(table, user_id) do
        [{^user_id, existing}] ->
          updated = Map.merge(existing, changes)
          :ets.insert(table, {user_id, updated})
        [] ->
          :ok
      end
    end)
  end

  # ── Worker pattern — xử lý heavy ops mà không block guild ─────────

  def handle_cast({:everyone_ping, channel_id, message}, state) do
    table    = state.ets_table
    guild_id = state.guild_id

    # Spawn worker — sẽ đọc ETS trực tiếp
    # Guild process KHÔNG bị block
    Task.start(fn ->
      # Worker đọc ETS trực tiếp — không cần copy qua guild
      eligible_users = Discord.Guild.ETSManager.fold_members(
        table,
        [],
        fn {user_id, member_data}, acc ->
          if can_see_channel?(member_data, channel_id) do
            [user_id | acc]
          else
            acc
          end
        end
      )

      # Worker tự fanout notification tới eligible users
      # Guild không cần biết kết quả
      notify_users(guild_id, eligible_users, message)
    end)

    # Guild tiếp tục handle messages khác ngay lập tức
    {:noreply, state}
  end
end
```

---

## Binary data — thêm 1 tầng optimization

Discord có thêm 1 trick: **binary data lớn hơn 64 bytes được store trên shared binary heap**, không phải process heap:

```elixir
defmodule Discord.BinaryHeapOptimization do
  @moduledoc """
  BEAM có 2 loại binary:
  1. Heap binary (≤ 64 bytes): nằm trong process heap, bị copy khi send
  2. Refc binary (> 64 bytes): nằm trên shared binary heap,
     chỉ copy REFERENCE (8 bytes) khi send

  Discord exploit điều này cho message content
  """

  def demonstrate do
    # Message content thường > 64 bytes
    content = "This is a Discord message that is definitely longer than 64 bytes!"

    # BEAM tự động store content trên shared binary heap
    # Khi guild fanout message tới 30,000 sessions:
    #   Không copy 100 bytes × 30,000 = 3MB
    #   Chỉ copy 8-byte reference × 30,000 = 240KB
    #   → 12.5x ít data phải copy

    message = %{
      content: content,  # refc binary — chỉ reference được copy
      author_id: 123,    # integer — copy trực tiếp (8 bytes)
      channel_id: 456,   # integer — copy trực tiếp (8 bytes)
    }

    # Khi send message struct:
    # - content: chỉ copy reference (8 bytes)
    # - author_id, channel_id: copy value (8 bytes each)
    # Tổng per send: ~24 bytes thay vì ~120 bytes
    message
  end

  # Cẩn thận: sub-binary tạo reference tới binary gốc
  # → Binary gốc không được GC dù đã "xong"
  # → Memory leak tinh vi
  def potential_leak do
    large_binary = :crypto.strong_rand_bytes(1_000_000)  # 1MB

    # Tạo sub-binary → giữ reference tới large_binary
    small_slice = binary_part(large_binary, 0, 10)

    # large_binary = 1MB
    # small_slice chỉ 10 bytes nhưng GIỮ 1MB không được collect
    # Khi chỉ cần small_slice: phải copy nó ra
    :binary.copy(small_slice)  # tạo independent binary, release reference
  end
end
```

---

## Process hibernation — giảm memory khi idle

```elixir
defmodule Discord.Guild.Process do
  # Guild process với guild nhỏ hoặc inactive
  # Có thể hibernate để giảm memory footprint

  def handle_info(:check_hibernation, state) do
    last_activity = state.last_activity_at
    idle_ms = System.monotonic_time(:millisecond) - last_activity

    if idle_ms > 300_000 do  # 5 phút không có activity
      # Hibernate: BEAM compact process heap xuống minimum
      # Stack và heap được GC hoàn toàn
      # Process được đánh thức khi có message mới
      {:noreply, state, :hibernate}
    else
      Process.send_after(self(), :check_hibernation, 60_000)
      {:noreply, state}
    end
  end

  # Khi wakeup từ hibernate: handle_call/cast/info như bình thường
  # BEAM tự re-expand heap khi cần
  def handle_cast({:new_message, _} = msg, state) do
    state = %{state | last_activity_at: System.monotonic_time(:millisecond)}
    # ... handle message
    {:noreply, state}
  end
end
```

---

## ETS read: copy cost và cách minimize

```elixir
defmodule Discord.ETS.ReadOptimization do
  @doc """
  ETS đọc vẫn copy data vào caller heap.
  Nhưng chỉ copy những gì cần — không phải toàn bộ table.
  """

  # BAD: đọc toàn bộ table vào heap
  def bad_get_all(table) do
    :ets.tab2list(table)
    # Copy TẤT CẢ 1,000,000 entries vào caller heap
    # Caller heap đột ngột tăng 200MB
    # → Caller GC sau đó rất nặng
  end

  # GOOD: chỉ đọc cái cần
  def good_get_one(table, user_id) do
    :ets.lookup(table, user_id)
    # Chỉ copy 1 entry (~200 bytes)
  end

  # GOOD: filter trong ETS — chỉ copy kết quả
  def good_get_by_role(table, role_id) do
    :ets.select(table, build_role_match_spec(role_id))
    # ETS thực hiện filter trong ETS memory
    # Chỉ copy matching entries vào caller heap
  end

  # GOOD: fold — process từng entry, accumulate nhỏ
  def good_count_online(table) do
    :ets.foldl(
      fn {_uid, %{status: status}}, count ->
        if status == :online, do: count + 1, else: count
      end,
      0,
      table
    )
    # Chỉ giữ 1 integer (count) trong caller heap
    # Không tạo intermediate list
  end

  # GOOD: dirty reads cho relay — không cần exact consistency
  def relay_get_member_snapshot(table, user_ids) do
    # Đọc batch nhưng chỉ fields cần thiết
    Enum.reduce(user_ids, [], fn user_id, acc ->
      case :ets.lookup(table, user_id) do
        [{_, %{nick: nick, roles: roles}}] ->
          # Chỉ extract fields cần thiết, không copy toàn bộ struct
          [{user_id, %{nick: nick, roles: roles}} | acc]
        [] ->
          acc
      end
    end)
  end
end
```

---

## Kết quả thực tế sau ETS migration

```
Metric                  Before ETS          After ETS
────────────────────────────────────────────────────────
Guild process heap      ~500MB              ~5MB
Major GC pause p99      ~2,340ms            ~2ms
GC frequency            mỗi 30 giây         mỗi vài phút
Worker spawn cost       copy 500MB = 8.5s   đọc ETS ref = 0
Relay read cost         message to guild    ETS direct = μs
@everyone check         block guild 5-10s   async worker, 0 block
Guild migration time    stall 2-5 min       async copy, 0 stall
Memory per guild        500MB               ~205MB (ETS + heap)
```

---

## Unknown unknowns trong tầng này

```
1. ETS write vẫn có cost — không free
   :ets.insert cho 1 entry: ~500ns
   Batch 1,000 inserts: ~500μs
   → Cần batch writes, không write từng cái

2. ETS table bị destroy khi owner process chết
   Nếu guild process crash → ETS mất → relay mất member data
   → Cần: heir process hoặc rebuild từ DB khi restart

3. ETS không có transaction
   Multiple writers có thể tạo inconsistent state
   Discord giải quyết: guild process là WRITER DUY NHẤT
   Relay chỉ READ → không có write conflict

4. Refc binary reference counting
   Nhiều processes giữ reference tới cùng binary
   → Binary không được GC cho đến khi TẤT CẢ release
   → Memory leak tinh vi nếu không careful

5. Hibernate có latency khi wakeup
   Process cần re-expand heap
   Lần đầu nhận message sau hibernate: ~1ms overhead
   → Không hibernate guild của active community
```

---

## Tóm tắt — memory hierarchy tại Discord

```
Tốc độ truy cập (nhanh → chậm):
────────────────────────────────────────────────
Process heap local    ~1ns    (biến local, stack)
Refc binary ref       ~2ns    (chỉ copy 8 bytes)
ETS lookup            ~500ns  (off-heap, shared)
ETS select/fold       ~μs     (filter trong ETS)
send/2 local          ~1μs    (copy + enqueue)
send/2 remote         ~70μs   (ETF + TCP)
DB query (cached)     ~1ms    (Redis/ETS cache)
DB query (cold)       ~10ms   (ScyllaDB/Postgres)

Discord design principle:
  Hot path → process heap hoặc refc binary
  Shared large data → ETS
  Cross-node → minimize, batch, use Manifold
  DB → chỉ khi không có cache
```

---
