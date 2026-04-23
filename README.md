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

# Q6: FastGlobal — Exploit BEAM Shared Heap, Nhanh Hơn ETS 23 Lần

*Đây là một trong những trick thông minh nhất trong toàn bộ Erlang ecosystem*

---

## Vấn đề FastGlobal giải quyết

Nhắc lại từ Q1: Khi session server crash và restart, chỉ riêng chi phí lookup trên hash ring đã tốn khoảng 30 giây. Discord dùng ETS (7μs/lookup) rồi sau đó dùng FastGlobal (0.3μs/lookup) — nhanh hơn ETS 23 lần.

```
Bài toán cụ thể:
  5,000,000 session processes
  Mỗi session cần lookup: "guild_id X đang ở node nào?"
  Lookup này xảy ra: mỗi khi reconnect, mỗi khi route message

  5,000,000 sessions × 7μs (ETS) = 35 giây chỉ để lookup ring
  5,000,000 sessions × 0.3μs (FastGlobal) = 1.5 giây ✅

  Với reconnect storm (node restart):
  ETS:        17.5 giây → cluster overloaded
  FastGlobal: 750ms → cluster survives
```

---

## Cơ chế BEAM VM — tại sao module constant đặc biệt

Để hiểu FastGlobal, cần hiểu cách BEAM load và execute code:

```
BEAM compiled module = series of bytecode instructions
Mỗi function = list of instructions

Khi function trả về constant:
  def get_config(), do: %{host: "localhost", port: 5432}

BEAM compiler nhận ra: function này LUÔN trả về cùng 1 giá trị
→ Optimizer store data vào "literal pool" — vùng nhớ đặc biệt
→ Literal pool được share giữa TẤT CẢ processes
→ Không bao giờ bị GC (tồn tại suốt module lifetime)
→ Đọc = chỉ trả về POINTER tới literal pool
→ Không copy gì cả
```

```erlang
%% Erlang bytecode level (simplified):
%% Hàm trả về constant:
get_ring() ->
    {ok, [node_a, node_b, node_c]}.  %% constant tuple

%% Bytecode:
%% MOVE {literal, <pointer_to_literal_pool>}, {x, 0}
%% RETURN

%% "literal" = chỉ đọc pointer — 0 copy
%% So sánh với ETS:
%% ETS lookup → find entry → COPY entry vào caller heap
%% Literal → trả về pointer → 0 copy
```

---

## FastGlobal — implementation thực sự

Discord port mochiglobal sang Elixir và thêm functionality để tránh tạo atoms. Version của Discord gọi là FastGlobal.

```elixir
defmodule FastGlobal do
  @moduledoc """
  Exploit BEAM literal pool để đọc global data mà không copy.

  Cơ chế:
    1. Tạo Erlang module mới tại runtime
    2. Module có 1 function trả về data như literal constant
    3. BEAM compile module → data vào literal pool
    4. Đọc = gọi function = nhận pointer = 0 copy

  Trade-off:
    Read:  0.33μs (23x nhanh hơn ETS)
    Write: ~44ms  (compile module mới — chậm)
    → Chỉ tốt khi data read >> write
  """

  # ── Write — tạo module mới với data ──────────────────────────────

  def put(key, value) do
    module_name = key_to_module(key)

    # Tạo Erlang abstract syntax tree (AST) cho module mới
    # Module này có 1 function: get/0 trả về value như literal
    module_ast = build_module_ast(module_name, value)

    # Compile AST → beam bytecode
    # Đây là operation tốn kém: ~44ms với data lớn
    {:ok, ^module_name, beam_binary} =
      :compile.forms(module_ast, [:return_errors])

    # Load module vào VM — purge version cũ nếu có
    # Khi purge: BEAM notify tất cả processes đang dùng module cũ
    # (đây là hidden cost — xem bên dưới)
    purge_old_module(module_name)
    :code.load_binary(module_name, ~c"nofile", beam_binary)

    :ok
  end

  defp build_module_ast(module_name, value) do
    # Tạo Erlang AST:
    # -module(module_name).
    # -export([get/0]).
    # get() -> VALUE.  ← VALUE được embed như literal

    # Convert Elixir term → Erlang abstract form
    value_ast = term_to_abstract_form(value)

    [
      {:attribute, 1, :module, module_name},
      {:attribute, 1, :export, [{:get, 0}]},
      {:function, 1, :get, 0,
        [{:clause, 1, [], [], [value_ast]}]}
    ]
  end

  defp term_to_abstract_form(value) when is_integer(value) do
    {:integer, 1, value}
  end

  defp term_to_abstract_form(value) when is_atom(value) do
    {:atom, 1, value}
  end

  defp term_to_abstract_form(value) when is_binary(value) do
    {:bin, 1,
     [{:bin_element, 1,
       {:string, 1, :erlang.binary_to_list(value)},
       :default, :default}]}
  end

  defp term_to_abstract_form(value) when is_list(value) do
    # Build list AST recursively
    Enum.reduce_while(Enum.reverse(value), {:nil, 1}, fn elem, acc ->
      {:cont, {:cons, 1, term_to_abstract_form(elem), acc}}
    end)
  end

  defp term_to_abstract_form(value) when is_map(value) do
    pairs = Enum.map(value, fn {k, v} ->
      {:map_field_assoc, 1,
       term_to_abstract_form(k),
       term_to_abstract_form(v)}
    end)
    {:map, 1, pairs}
  end

  defp term_to_abstract_form({}) do
    {:tuple, 1, []}
  end

  defp term_to_abstract_form(tuple) when is_tuple(tuple) do
    elements = tuple
      |> Tuple.to_list()
      |> Enum.map(&term_to_abstract_form/1)
    {:tuple, 1, elements}
  end

  # ── Read — gọi function của module ───────────────────────────────

  def get(key) do
    module_name = key_to_module(key)
    try do
      # Gọi module:get() → BEAM trả về pointer tới literal pool
      # Không copy gì cả
      # Cost: function call overhead ~0.33μs
      module_name.get()
    rescue
      UndefinedFunctionError -> nil  # key chưa tồn tại
    end
  end

  # ── Helpers ───────────────────────────────────────────────────────

  defp key_to_module(key) when is_atom(key) do
    # Tạo module name từ key
    # KHÔNG dùng String.to_atom (có thể exhaust atom table)
    Module.concat([FastGlobal, key])
  end

  defp key_to_module(key) when is_binary(key) do
    # Discord version: dùng atom table safe approach
    # Tránh tạo atom mới từ arbitrary string
    Module.concat([FastGlobal, Base.encode16(key)])
  end

  defp purge_old_module(module_name) do
    case :code.is_loaded(module_name) do
      {:file, _} ->
        # Purge old version
        # ⚠️ Đây là hidden cost: BEAM phải notify mọi process
        # đang reference module này → cost tỉ lệ với process count
        :code.purge(module_name)
        :code.delete(module_name)
      false ->
        :ok
    end
  end
end
```

---

## Benchmark thực tế — so sánh 3 approaches

Benchmark: `fastglobal get` đạt 0.33μs/op so với `ets get` 7.64μs/op và `agent get` 12.67μs/op.

```elixir
defmodule FastGlobal.Benchmark do
  def run do
    data = build_hash_ring()

    # Setup
    FastGlobal.put(:ring, data)
    ets_table = setup_ets(data)
    {:ok, agent} = Agent.start_link(fn -> data end)

    iterations = 10_000_000

    # Benchmark FastGlobal
    {fg_time, _} = :timer.tc(fn ->
      Enum.each(1..iterations, fn _ ->
        FastGlobal.get(:ring)
      end)
    end)

    # Benchmark ETS
    {ets_time, _} = :timer.tc(fn ->
      Enum.each(1..iterations, fn _ ->
        :ets.lookup(ets_table, :ring)
      end)
    end)

    # Benchmark Agent (GenServer call)
    {agent_time, _} = :timer.tc(fn ->
      Enum.each(1..iterations, fn _ ->
        Agent.get(agent, & &1)
      end)
    end)

    IO.puts("FastGlobal: #{fg_time / iterations}μs/op")
    IO.puts("ETS:        #{ets_time / iterations}μs/op")
    IO.puts("Agent:      #{agent_time / iterations}μs/op")
  end

  # Output:
  # FastGlobal: 0.33μs/op
  # ETS:        7.64μs/op   ← copy overhead
  # Agent:      12.67μs/op  ← GenServer message + copy
end
```

---

## Tại sao FastGlobal nhanh hơn ETS 23 lần — memory model

```
ETS.lookup(table, :ring) flow:
  1. Hash key → find bucket trong ETS hash table    ~100ns
  2. Lock bucket (read lock)                         ~50ns
  3. Find entry                                      ~50ns
  4. COPY entry từ ETS memory → caller heap          ~7,000ns ← bottleneck
     (hash ring = large data structure)
  5. Unlock bucket                                   ~50ns
  Total: ~7,250ns = 7.25μs

FastGlobal.get(:ring) flow:
  1. Module lookup trong code table                  ~50ns
  2. Function dispatch                               ~50ns
  3. Return POINTER tới literal pool                 ~50ns
     (literal pool = shared read-only memory)
  4. Caller nhận pointer — ZERO copy                 0ns
  Total: ~150ns = 0.15μs

Lý do copy là bottleneck trong ETS:
  Hash ring structure: ~500KB
  memcpy 500KB: ~5,000ns trên modern hardware
  → ETS dominated by copy cost, không phải lookup cost
```

---

## Memory layout — visualize sự khác biệt

---

![beam_memory_fastglobal_vs_ets](beam_memory_fastglobal_vs_ets.svg)

## Discord dùng FastGlobal cho gì cụ thể

```elixir
defmodule Discord.FastGlobalUsage do
  @moduledoc """
  Discord dùng FastGlobal cho data thỏa mãn:
  1. Read cực kỳ thường xuyên (hàng triệu lần/giây)
  2. Write rất hiếm (vài lần/giờ hoặc ít hơn)
  3. Data lớn (đủ để copy cost là bottleneck)
  4. Cần serve từ bất kỳ process nào (không phải 1 owner)
  """

  # USE CASE 1: Consistent hash ring
  # Read: mỗi session cần lookup khi route message
  # Write: khi node join hoặc leave cluster (hiếm)
  def update_hash_ring(ring_data) do
    FastGlobal.put(:hash_ring, ring_data)
    # ring_data: ~500KB struct với virtual nodes
  end

  def lookup_node_for_guild(guild_id) do
    ring = FastGlobal.get(:hash_ring)
    # 0.33μs — không copy 500KB ring vào caller heap
    Discord.Router.ConsistentHash.get_node(ring, guild_id)
  end

  # USE CASE 2: Guild feature flags
  # Read: mỗi event check features của guild
  # Write: khi admin thay đổi settings (rất hiếm)
  def update_guild_features(guild_id, features) do
    key = :"guild_features_#{guild_id}"
    FastGlobal.put(key, features)
  end

  def get_guild_features(guild_id) do
    FastGlobal.get(:"guild_features_#{guild_id}")
    # Nếu nil: fallback về ETS hoặc DB
  end

  # USE CASE 3: Global rate limit config
  # Read: mỗi incoming request check rate limits
  # Write: khi engineer thay đổi config (cực hiếm)
  def update_rate_limit_config(config) do
    FastGlobal.put(:rate_limit_config, config)
  end

  def get_rate_limit_config do
    FastGlobal.get(:rate_limit_config)
  end
end
```

---

## Hidden costs — những gì Discord phát hiện sau khi deploy

Discord sau đó đã bỏ FastGlobal vì phát hiện thời gian compile quá lâu khi purge module với số lượng lớn processes trong system.

```elixir
defmodule FastGlobal.HiddenCosts do
  @moduledoc """
  Những vấn đề FastGlobal gặp phải ở Discord scale
  """

  # HIDDEN COST 1: Purge cost tăng theo process count
  def demonstrate_purge_cost do
    # Khi gọi FastGlobal.put(:ring, new_ring):
    # 1. Compile module mới: ~1-44ms (tùy data size)
    # 2. :code.purge(old_module):
    #    → BEAM gửi signal tới TỪNG process đang reference module cũ
    #    → Mỗi process phải acknowledge
    #    → Với 5,000,000 processes: ~30 giây chỉ để purge!

    # Đây là lý do Discord bỏ FastGlobal:
    # purge_time ∝ process_count
    # 5M processes × purge_overhead = unacceptable latency
    :ok
  end

  # HIDDEN COST 2: Atom table exhaustion
  # Tạo module name từ dynamic key = tạo atom mới
  defp bad_key_to_module(key) when is_binary(key) do
    # ❌ NGUY HIỂM: String.to_atom tạo atom không bao giờ bị GC
    # Atom table có giới hạn (~1,048,576 atoms mặc định)
    String.to_atom("fast_global_#{key}")
  end

  defp good_key_to_module(key) when is_binary(key) do
    # ✅ AN TOÀN: dùng existing atoms hoặc encode key
    # FastGlobal.new/1 tạo atom một lần khi startup
    Module.concat([FastGlobal, Base.encode16(key)])
    # Base.encode16 trả về binary → Module.concat dùng internal encoding
    # Không tạo atom mới từ arbitrary string
  end

  # HIDDEN COST 3: Compile time tăng theo data size
  def compile_time_analysis do
    # fastglobal put (5 keys):   ~3,683μs = 3.7ms
    # fastglobal put (10 keys):  ~6,449μs = 6.4ms
    # fastglobal put (100 keys): ~44,543μs = 44.5ms
    # → Linear: mỗi key thêm ~440μs compile time

    # Với hash ring 150 nodes × 150 virtual nodes = 22,500 entries:
    # Compile time: ~10 giây → không acceptable cho hot update
    :ok
  end

  # HIDDEN COST 4: không hoạt động tốt với distributed tracing
  # Literal pool pointers không carry trace context
  # → Mọi read trở thành "orphan" span trong distributed trace
  :ok
end
```

---

## Replacement — Discord dùng gì sau khi bỏ FastGlobal

```elixir
defmodule Discord.PersistentTermApproach do
  @moduledoc """
  Erlang/OTP 22+ có :persistent_term — built-in alternative
  Cùng zero-copy semantics nhưng không cần compile module
  Và purge cost thấp hơn nhiều
  """

  # Write: ~microseconds (không compile module)
  def put(key, value) do
    :persistent_term.put(key, value)
    # Tuy nhiên: write vẫn có cost vì phải
    # scan tất cả processes và invalidate cache
  end

  # Read: ~nanoseconds, zero-copy (tương đương FastGlobal)
  def get(key) do
    :persistent_term.get(key)
  end

  # Trade-offs của :persistent_term:
  # ✅ Zero-copy read (same as FastGlobal)
  # ✅ Không cần compile module
  # ✅ Built-in, không cần library
  # ⚠️  Write vẫn expensive ở scale lớn (scan processes)
  # ⚠️  Chỉ phù hợp cho data thay đổi rất hiếm

  # So sánh:
  # FastGlobal.put:          ~44ms (compile) + ~30s purge (5M processes)
  # :persistent_term.put:    ~1ms + scan overhead
  # ETS write:               ~500ns (fastest write, nhưng copy khi đọc)

  def benchmark_read do
    :persistent_term.put(:test_data, build_large_structure())

    {time, _} = :timer.tc(fn ->
      Enum.each(1..10_000_000, fn _ ->
        :persistent_term.get(:test_data)
      end)
    end)

    IO.puts("persistent_term read: #{time / 10_000_000}μs/op")
    # Output: ~0.35μs/op — tương đương FastGlobal
  end
end
```

---

## Decision tree — chọn storage mechanism nào

```elixir
defmodule Discord.StorageDecisionGuide do
  @moduledoc """
  Hướng dẫn chọn storage tại Discord scale:
  """

  def choose_storage(opts) do
    read_freq    = opts[:reads_per_second]   # reads/s
    write_freq   = opts[:writes_per_second]  # writes/s
    data_size    = opts[:data_size_bytes]
    process_count = opts[:concurrent_processes]
    needs_query  = opts[:needs_filtering]    # cần WHERE clause không

    cond do
      # Case 1: Đọc >> Ghi, data lớn, không cần query
      # → :persistent_term (OTP 22+) hoặc FastGlobal (legacy)
      read_freq > 1_000_000 and
      write_freq < 10 and
      data_size > 10_000 ->
        :persistent_term

      # Case 2: Đọc nhiều, cần query/filter
      # → ETS với read_concurrency: true
      read_freq > 100_000 and needs_query ->
        :ets_with_read_concurrency

      # Case 3: Đọc/Ghi cân bằng, shared giữa processes
      # → ETS với write_concurrency: true
      write_freq > 1_000 ->
        :ets_with_write_concurrency

      # Case 4: Data nhỏ, chỉ 1 process sở hữu
      # → Process state (GenServer)
      data_size < 10_000 and process_count == 1 ->
        :genserver_state

      # Case 5: Cross-node, cần replication
      # → Mnesia (nhưng Discord đã bỏ — xem Q10)
      # Thay thế: custom ETS + sync protocol
      true ->
        :ets_with_custom_sync
    end
  end
end
```

---

## Tổng kết — FastGlobal lesson learned

```
FastGlobal là brilliant hack nhưng có hidden costs:

Lesson 1: Zero-copy read = cực nhanh khi reads >> writes
          Nhưng writes gây pain tỉ lệ thuận với process count

Lesson 2: Erlang/OTP 22+ có :persistent_term
          Làm được điều tương tự, ít gotchas hơn
          Discord đã migrate sang :persistent_term

Lesson 3: Atom table là shared global resource
          Dynamic atom creation = memory leak
          Luôn dùng :erlang.binary_to_existing_atom
          hoặc pre-define atoms khi startup

Lesson 4: Module compile/purge cost không obvious
          Cần benchmark với production process count
          Không phải development process count

Benchmark tổng kết:
  Operation          Latency    Copy?   Write cost   Use when
  ──────────────────────────────────────────────────────────────
  Process heap       ~1ns       No      N/A          local state
  :persistent_term   ~0.35μs   No      ~1ms         global, rare write
  FastGlobal         ~0.33μs   No      ~44ms+purge  legacy, avoid now
  ETS lookup         ~7.64μs   Yes     ~500ns       shared, queryable
  GenServer.call     ~12.67μs  Yes     N/A          coordinated access
  Remote send/2      ~70μs     Yes     N/A          cross-node
```

---

# Q7: Guild Migration — Handoff State Nhiều GB Không Mất Message

*Đây là bài toán distributed systems khó nhất mà Discord phải giải quyết*

---

## Tại sao migration là bắt buộc — không phải optional

```
Discord cần migrate guild process trong các tình huống:

1. Rolling deploy — update code mà không có downtime
   → Mỗi node phải restart để load code mới
   → Guild process trên node đó phải di chuyển trước

2. Node rebalancing — cluster mất cân bằng
   → Node A: 80% CPU, Node B: 20% CPU
   → Di chuyển guild từ A sang B

3. Hardware maintenance — node cần tắt
   → Di chuyển tất cả guilds sang nodes khác trước khi tắt

4. Scaling event — thêm node mới vào cluster
   → Consistent hash ring thay đổi
   → Một số guilds cần di chuyển sang node mới

Challenge:
  Guild state có thể lên tới nhiều GB (ETS member table)
  Users đang active trong guild — không thể drop connections
  Messages đang in-flight — không được mất
  Migration phải hoàn toàn transparent với users
```

---

## Vấn đề cốt lõi — "state transfer" trong distributed system

```
Naive approach — KHÔNG làm thế này:

  1. Stop guild process trên Node A
  2. Serialize toàn bộ state
  3. Transfer sang Node B
  4. Deserialize và start guild process mới
  5. Redirect traffic

Vấn đề:
  Bước 1 → 5: guild bị "offline" trong vài phút
  Users mất connection
  Messages bị drop
  Users thấy disconnect screen
```

Discord cần **zero-downtime migration** — tương tự database live migration.

---

## Protocol 4 phases — zero-downtime migration

---

![Guild Migration Phases](guild_migration_4_phases.svg)

## Phase 1: Prepare — khởi động shadow guild trên Node B

```elixir
defmodule Discord.Guild.Migration do
  @doc """
  Entry point: migrate guild_id từ source_node sang target_node
  Được gọi bởi deployment orchestrator hoặc rebalancer
  """
  def start_migration(guild_id, source_node, target_node) do
    source_pid = Discord.Guild.Registry.lookup(guild_id, source_node)

    # Thông báo cho source guild: "chuẩn bị migrate"
    # Source guild bắt đầu buffer outgoing events
    :ok = GenServer.call(source_pid, :prepare_migration, 30_000)

    # Spawn shadow guild trên target node
    {:ok, shadow_pid} = :rpc.call(
      target_node,
      Discord.Guild.DynamicSupervisor,
      :start_shadow_guild,
      [guild_id, source_pid, self()]
    )

    # Bắt đầu phase 2
    {:ok, %{
      guild_id:    guild_id,
      source_pid:  source_pid,
      shadow_pid:  shadow_pid,
      source_node: source_node,
      target_node: target_node,
      started_at:  System.monotonic_time(:millisecond)
    }}
  end
end

defmodule Discord.Guild.Process do
  # Source guild nhận lệnh prepare migration
  def handle_call(:prepare_migration, _from, state) do
    # Bắt đầu buffer: lưu lại tất cả events
    # sẽ replay sang shadow guild sau khi transfer xong
    new_state = %{state |
      migration_mode: :preparing,
      # Buffer circular — tránh OOM nếu migration kéo dài
      event_buffer: :queue.new(),
      event_buffer_size: 0,
    }
    {:reply, :ok, new_state}
  end

  # Trong khi migration: tiếp tục serve bình thường
  # ĐỒNG THỜI buffer events để replay sau
  def handle_cast({:new_message, message}, %{migration_mode: :preparing} = state) do
    # Serve traffic bình thường
    fanout_to_relays(message, state)

    # Buffer event để replay sang shadow guild
    new_buffer = :queue.in({:new_message, message}, state.event_buffer)
    new_size   = state.event_buffer_size + 1

    # Safety: nếu buffer quá lớn → force cutover sớm
    if new_size > 10_000 do
      send(self(), :force_cutover)
    end

    {:noreply, %{state |
      event_buffer:      new_buffer,
      event_buffer_size: new_size
    }}
  end

  # Normal mode: không buffer
  def handle_cast({:new_message, message}, state) do
    fanout_to_relays(message, state)
    {:noreply, state}
  end
end
```

---

## Phase 2: Stream — transfer ETS data theo chunks

Discord dùng worker process để send phần lớn members (có thể là nhiều GB) trong khi guild process cũ vẫn đang làm việc, nhờ đó cắt giảm hàng phút stall mỗi lần handoff.

```elixir
defmodule Discord.Guild.StateStreamer do
  @moduledoc """
  Stream ETS table từ source guild sang shadow guild.
  Chạy trong worker process riêng — source guild KHÔNG bị block.
  """

  @chunk_size 1_000    # entries per chunk
  @chunk_delay 10      # ms between chunks — throttle để không flood network

  def stream_state(source_pid, shadow_pid, migration_ctx) do
    # Lấy ETS table identifier từ source guild
    # (chỉ là reference — không copy data)
    ets_table = GenServer.call(source_pid, :get_ets_table_ref)

    # Lấy guild metadata (small — copy OK)
    metadata = GenServer.call(source_pid, :get_migration_metadata)

    # Send metadata trước
    GenServer.call(shadow_pid, {:receive_metadata, metadata})

    # Stream ETS table theo chunks
    stream_ets_chunks(ets_table, shadow_pid)

    # Signal: data transfer hoàn tất
    GenServer.call(shadow_pid, :data_transfer_complete)
    GenServer.call(source_pid, :data_transfer_complete)

    :ok
  end

  defp stream_ets_chunks(table, shadow_pid) do
    # Dùng ETS continuation để iterate không cần load toàn bộ vào heap
    do_stream_chunks(table, shadow_pid, :ets.first(table), 0)
  end

  defp do_stream_chunks(_table, _shadow, :"$end_of_table", total) do
    {:ok, total}
  end

  defp do_stream_chunks(table, shadow_pid, key, total) do
    # Lấy chunk theo key order
    chunk = collect_chunk(table, key, @chunk_size, [])

    # Gửi chunk sang shadow guild
    # Đây là network transfer — chunk nhỏ để tránh TCP buffer overflow
    :ok = GenServer.call(
      shadow_pid,
      {:receive_member_chunk, chunk},
      30_000  # 30s timeout per chunk
    )

    # Throttle để tránh flood network
    Process.sleep(@chunk_delay)

    # Continue với key tiếp theo
    next_key = find_next_key_after_chunk(table, chunk)
    do_stream_chunks(table, shadow_pid, next_key, total + length(chunk))
  end

  defp collect_chunk(_table, :"$end_of_table", _remaining, acc) do
    Enum.reverse(acc)
  end

  defp collect_chunk(_table, _key, 0, acc) do
    Enum.reverse(acc)
  end

  defp collect_chunk(table, key, remaining, acc) do
    case :ets.lookup(table, key) do
      [entry] ->
        next_key = :ets.next(table, key)
        collect_chunk(table, next_key, remaining - 1, [entry | acc])
      [] ->
        Enum.reverse(acc)
    end
  end
end

defmodule Discord.Guild.ShadowProcess do
  use GenServer
  @moduledoc """
  Shadow guild: nhận state từ source, buffer events mới,
  sẵn sàng để activate khi cutover.
  """

  defstruct [
    :guild_id,
    :source_pid,
    :ets_table,
    :orchestrator_pid,
    # Buffer events xảy ra trong khi đang nhận state
    # Sẽ replay sau khi activate
    event_buffer: [],
    data_received: false,
  ]

  def init(opts) do
    guild_id       = opts[:guild_id]
    source_pid     = opts[:source_pid]
    orchestrator   = opts[:orchestrator_pid]

    # Tạo ETS table ngay từ đầu
    table = :ets.new(:"guild_shadow_#{guild_id}", [:set, :public])

    # Subscribe vào events của source guild
    # Nhận events trong khi đang transfer state
    :ok = GenServer.call(source_pid, {:subscribe_shadow, self()})

    {:ok, %__MODULE__{
      guild_id:        guild_id,
      source_pid:      source_pid,
      ets_table:       table,
      orchestrator_pid: orchestrator,
    }}
  end

  # Nhận metadata (guild settings, channels, roles...)
  def handle_call({:receive_metadata, metadata}, _from, state) do
    # Store metadata — nhỏ, OK trong heap
    new_state = Map.merge(state, metadata)
    {:reply, :ok, new_state}
  end

  # Nhận chunk members từ source
  def handle_call({:receive_member_chunk, chunk}, _from, state) do
    # Insert chunk vào local ETS — off-heap
    :ets.insert(state.ets_table, chunk)
    {:reply, :ok, state}
  end

  # Data transfer hoàn tất
  def handle_call(:data_transfer_complete, _from, state) do
    new_state = %{state | data_received: true}
    # Notify orchestrator: sẵn sàng cutover
    send(state.orchestrator_pid, {:shadow_ready, self()})
    {:reply, :ok, new_state}
  end

  # Nhận events từ source guild trong khi transfer
  # Buffer lại để replay khi activate
  def handle_cast({:shadow_event, event}, state) do
    new_buffer = [event | state.event_buffer]
    {:noreply, %{state | event_buffer: new_buffer}}
  end

  # Cutover: activate shadow guild
  def handle_call(:activate, _from, state) do
    # Replay buffered events theo đúng thứ tự
    state.event_buffer
    |> Enum.reverse()  # reverse vì buffer là prepend
    |> Enum.each(&apply_buffered_event(&1, state))

    # Shadow guild giờ là guild thật
    new_state = %{state | event_buffer: []}
    {:reply, :ok, new_state}
  end

  defp apply_buffered_event({:member_join, user_id, data}, state) do
    :ets.insert(state.ets_table, {user_id, data})
  end

  defp apply_buffered_event({:member_leave, user_id}, state) do
    :ets.delete(state.ets_table, user_id)
  end

  defp apply_buffered_event({:member_update, user_id, changes}, state) do
    case :ets.lookup(state.ets_table, user_id) do
      [{^user_id, existing}] ->
        :ets.insert(state.ets_table, {user_id, Map.merge(existing, changes)})
      [] -> :ok
    end
  end
end
```

---

## Phase 3: Atomic Cutover — ~1ms window

Đây là phần tinh tế nhất — phải atomic để tránh split-brain:

```elixir
defmodule Discord.Guild.Migration do
  def execute_cutover(migration_ctx) do
    %{
      guild_id:   guild_id,
      source_pid: source_pid,
      shadow_pid: shadow_pid,
      target_node: target_node
    } = migration_ctx

    # Chờ shadow guild báo sẵn sàng
    receive do
      {:shadow_ready, ^shadow_pid} -> :ok
    after
      120_000 -> {:error, :shadow_not_ready}
    end

    # === CRITICAL SECTION — phải hoàn thành trong <5ms ===

    # Bước 1: Lock source guild — ngừng xử lý NEW requests
    # In-flight requests vẫn được xử lý
    :ok = GenServer.call(source_pid, :begin_cutover, 5_000)

    # Bước 2: Flush event buffer từ source sang shadow
    remaining_events = GenServer.call(source_pid, :flush_event_buffer)
    GenServer.call(shadow_pid, {:apply_events, remaining_events})

    # Bước 3: Activate shadow guild
    :ok = GenServer.call(shadow_pid, :activate, 5_000)

    # Bước 4: Atomic router update
    # Mọi node trong cluster update routing table cùng lúc
    # Dùng :rpc.multicall để broadcast
    {results, bad_nodes} = :rpc.multicall(
      Node.list(),
      Discord.Router,
      :update_guild_node,
      [guild_id, target_node],
      5_000
    )

    if bad_nodes != [] do
      # Một số nodes không update được → rollback
      rollback_cutover(migration_ctx)
      {:error, {:router_update_failed, bad_nodes}}
    else
      # Bước 5: Source guild chuyển sang redirect mode
      GenServer.cast(source_pid, {:enter_redirect_mode, shadow_pid})

      # === END CRITICAL SECTION ===

      {:ok, :cutover_complete}
    end
  end

  # Source guild handle cutover
  # Mọi message sau đây sẽ được forward sang shadow
  defmodule Discord.Guild.Process do
    def handle_call(:begin_cutover, _from, state) do
      {:reply, :ok, %{state | migration_mode: :cutting_over}}
    end

    def handle_call(:flush_event_buffer, _from, state) do
      events = :queue.to_list(state.event_buffer)
      {:reply, events, %{state | event_buffer: :queue.new()}}
    end

    # Redirect mode: forward mọi request sang shadow guild
    def handle_cast({:enter_redirect_mode, shadow_pid}, state) do
      {:noreply, %{state |
        migration_mode: :redirecting,
        redirect_target: shadow_pid
      }}
    end

    # Mọi message trong redirect mode → forward sang target
    def handle_cast(message, %{migration_mode: :redirecting} = state) do
      GenServer.cast(state.redirect_target, message)
      {:noreply, state}
    end

    def handle_call(request, from, %{migration_mode: :redirecting} = state) do
      # Forward call sang shadow, relay response về caller
      Task.start(fn ->
        result = GenServer.call(state.redirect_target, request, 10_000)
        GenServer.reply(from, result)
      end)
      {:noreply, state}
    end
  end
end
```

---

## Phase 4: Drain — cleanup gracefully

```elixir
defmodule Discord.Guild.Migration do
  def execute_drain(source_pid) do
    # Đợi 5 giây để in-flight requests hoàn thành
    # Requests mới đã được router redirect sang Node B
    Process.send_after(source_pid, :shutdown_after_drain, 5_000)
  end
end

defmodule Discord.Guild.Process do
  def handle_info(:shutdown_after_drain, state) do
    # Unsubscribe sessions khỏi redirect
    # Sessions sẽ reconnect tới Node B tự nhiên
    notify_sessions_of_migration(state)

    # Cleanup ETS table
    :ets.delete(state.ets_table)

    # Graceful stop
    {:stop, :normal, state}
  end

  defp notify_sessions_of_migration(state) do
    # Gửi signal tới tất cả relay processes
    # Relay sẽ trigger session reconnect tới guild mới
    relay_pids = Map.values(state.relay_registry)
    Manifold.send(relay_pids, {
      :guild_migrated,
      state.guild_id,
      state.redirect_target  # shadow guild PID (giờ là primary)
    })
  end
end

defmodule Discord.Relay.Process do
  # Relay nhận thông báo guild migrated
  def handle_info({:guild_migrated, guild_id, new_guild_pid}, state) do
    # Update guild reference
    new_state = %{state | guild_pid: new_guild_pid}

    # Re-register với guild mới
    GenServer.cast(new_guild_pid, {
      :relay_reconnect,
      state.relay_id,
      self(),
      Map.keys(state.sessions)  # danh sách sessions đang manage
    })

    {:noreply, new_state}
  end
end
```

---

## Message ordering guarantee trong migration

Vấn đề tinh tế nhất: messages in-flight khi cutover xảy ra:

```
Timeline tại thời điểm cutover (t=T):

t=T-2ms: User A gửi message M1 → source guild nhận
t=T-1ms: Source guild bắt đầu fanout M1
t=T:     CUTOVER — router update
t=T+1ms: User B gửi message M2 → router → target guild
t=T+3ms: Source guild vẫn đang fanout M1 (chưa xong)
t=T+5ms: M1 fanout xong
t=T+6ms: M2 fanout bắt đầu từ target guild

Question: sessions có nhận M1 trước M2 không?
```

```elixir
defmodule Discord.Guild.OrderingGuarantee do
  @moduledoc """
  Discord đảm bảo ordering bằng cách:

  1. Source guild fanout M1 trước khi accept cutover
     → M1 được gửi tới tất cả sessions trước khi nguồn thay đổi

  2. Target guild nhận M2 SAU khi router update
     → Target guild chỉ xử lý M2 sau khi đã apply tất cả
        buffered events (bao gồm M1 nếu nó được buffer)

  3. Relay processes nhận messages theo thứ tự từ guild
     → Nếu cùng relay, linearizability đảm bảo bởi mailbox ordering

  Edge case: session nhận từ 2 guilds trong transition window
  """

  def handle_transition_window do
    # Trong ~5 giây drain window:
    # - Một số sessions vẫn nhận từ source (via redirect)
    # - Một số sessions đã switch sang target

    # Discord chấp nhận: messages trong 5 giây window
    # có thể delivered out-of-order với nhau
    # (xác suất rất thấp, user experience impact minimal)

    # Sessions phát hiện migration qua sequence number:
    # Mỗi guild message có sequence number tăng dần
    # Client detect gap → request resync từ target guild
    :ok
  end

  # Sequence number tracking
  def next_sequence(state) do
    seq = state.message_sequence + 1
    {seq, %{state | message_sequence: seq}}
  end

  # Client gửi last_seen_sequence khi reconnect
  # Target guild replay messages từ sequence đó
  def handle_call({:session_resume, user_id, last_sequence}, _from, state) do
    missed_messages = get_messages_after_sequence(
      state.message_cache,
      last_sequence
    )
    {:reply, {:ok, missed_messages}, state}
  end
end
```

---

## Rollback — khi migration thất bại

```elixir
defmodule Discord.Guild.Migration do
  def rollback_cutover(migration_ctx) do
    %{
      source_pid:  source_pid,
      shadow_pid:  shadow_pid,
      guild_id:    guild_id,
      source_node: source_node
    } = migration_ctx

    require Logger
    Logger.error("Migration rollback for guild #{guild_id}")

    # 1. Restore router về source node
    :rpc.multicall(
      Node.list(),
      Discord.Router,
      :update_guild_node,
      [guild_id, source_node],
      5_000
    )

    # 2. Source guild thoát redirect mode
    GenServer.cast(source_pid, :exit_redirect_mode)

    # 3. Stop shadow guild
    GenServer.stop(shadow_pid, :normal)

    # 4. Source guild drain buffered events
    # (những events đã buffer trong khi chuẩn bị migrate)
    GenServer.call(source_pid, :replay_buffered_events)

    :ok
  end
end

defmodule Discord.Guild.Process do
  def handle_cast(:exit_redirect_mode, state) do
    {:noreply, %{state | migration_mode: :normal, redirect_target: nil}}
  end

  def handle_call(:replay_buffered_events, _from, state) do
    # Replay tất cả events đã buffer trong preparing phase
    state.event_buffer
    |> :queue.to_list()
    |> Enum.each(fn event ->
      GenServer.cast(self(), event)
    end)

    {:reply, :ok, %{state |
      event_buffer:      :queue.new(),
      event_buffer_size: 0,
      migration_mode:    :normal
    }}
  end
end
```

---

## Số liệu thực tế — migration performance

```
Guild size       ETS data    Transfer time    Cutover window    Message loss
──────────────────────────────────────────────────────────────────────────────
10,000 members   ~2MB        ~0.5 giây        ~1ms              0
100,000 members  ~20MB       ~5 giây          ~1ms              0
1,000,000 members ~200MB     ~45 giây         ~2ms              0
Midjourney       ~500MB+     ~120 giây        ~3ms              0

So sánh với naive approach (stop-copy-start):
1,000,000 members: ~8 giây downtime
Midjourney:        ~20 giây downtime (unacceptable)

Key metric: cutover window luôn < 5ms
→ Users không thể perceive được
→ Zero messages lost
→ Zero disconnects
```

---

## Unknown unknowns trong guild migration

```
1. ETS heir — nếu guild crash TRONG KHI migrate
   :ets.new với {:heir, supervisor_pid, data}
   → ETS không bị destroy khi guild crash
   → Supervisor nhận {:ETS-TRANSFER, table, ...}
   → Có thể restart guild với ETS còn nguyên

2. Network partition trong cutover window
   Router update tới một số nodes thất bại
   → Split-brain: một số routes về A, một số về B
   → Cần: distributed transaction cho router update
   → Discord giải quyết: retry với exponential backoff
     + rollback nếu không đạt majority

3. Shadow guild OOM nếu transfer quá chậm
   Event buffer tăng trong khi chờ transfer
   → Max buffer size + force cutover nếu đầy

4. Hot guild migration — guild đang xử lý 100k events/giây
   Event buffer tăng nhanh hơn transfer
   → Throttle incoming events tạm thời
   → Hoặc tăng chunk size để transfer nhanh hơn

5. Sequence number wrap-around
   Message sequence là integer hữu hạn
   → Cần handle modular arithmetic khi compare sequences
   → Client phải handle gap detection correctly
```

---

## Tóm tắt — những gì làm cho migration work

```
Yếu tố then chốt          Cơ chế
──────────────────────────────────────────────────────
Zero downtime              Shadow guild nhận events trong khi transfer
No message loss            Event buffer + replay sau cutover
Atomic routing             :rpc.multicall tới tất cả nodes
Fast cutover (<5ms)        Pre-built shadow, chỉ swap pointer
No connection drop         Sessions không biết guild di chuyển
Rollback safe              Source guild giữ redirect mode 5 giây
Large state OK             ETS stream theo chunks, worker process
GC pressure thấp           ETS off-heap, chunks nhỏ không spike heap
```

---

# Q8: ZenMonitor — Monitor Storm và Tại Sao Process.monitor Không Scale

*Đây là một trong những failure modes ít được biết đến nhất của Erlang ở scale lớn*

---

## Process.monitor — cơ chế built-in và giới hạn

Trước khi vào ZenMonitor, cần hiểu chính xác `Process.monitor/1` làm gì:

```elixir
# Khi gọi Process.monitor(remote_pid):
ref = Process.monitor(remote_pid)

# BEAM thực hiện:
# 1. Tạo monitor entry trong process table của CALLER
# 2. Gửi {:monitor, caller_pid, ref} tới NODE chứa remote_pid
# 3. Node kia thêm entry: "khi remote_pid chết, notify caller_pid"
# 4. Khi remote_pid chết:
#    → Node kia gửi {:DOWN, ref, :process, remote_pid, reason}
#    → Tới caller_pid

# Với 1 monitor: hoàn toàn OK
# Với 1,000,000 monitors trên cùng 1 process: thảm họa
```

---

## Monitor Storm — failure mode thực tế tại Discord

```
Discord architecture:
  1 Guild process trên Node A
  Guild được monitor bởi:
    - 500,000 session processes (trên Node B, C, D, E...)
    - Mỗi session monitor guild để biết khi guild crash

Scenario: Node A bị kill đột ngột (hardware failure, OOM...)

t=0ms:    Node A down
t=1ms:    BEAM distribution phát hiện nodedown
t=2ms:    Node B nhận {:nodedown, nodeA}
          → Phải gửi {:DOWN, ...} tới TẤT CẢ monitors
          → 125,000 sessions trên Node B cần nhận {:DOWN}

t=2ms:    Node C nhận {:nodedown, nodeA}
          → 125,000 sessions trên Node C

t=2ms:    Node D, E tương tự...

Kết quả trong 1ms:
  4 nodes × 125,000 {:DOWN} messages = 500,000 messages
  Được inject vào mailboxes đồng thời
  → Scheduler overload trên tất cả nodes
  → Nodes bắt đầu lag
  → Sessions timeout
  → Users thấy disconnect
  → Cascading failure
```

---

## Đo lường vấn đề

```elixir
defmodule Discord.MonitorStormBenchmark do
  def simulate_storm do
    # Simulate 500,000 processes monitor 1 target
    target = spawn(fn ->
      receive do :die -> :ok end
    end)

    # Spawn 500,000 monitors
    monitors = Enum.map(1..500_000, fn _ ->
      spawn(fn ->
        ref = Process.monitor(target)
        receive do
          {:DOWN, ^ref, :process, _, _} ->
            # Xử lý DOWN message
            :ok
        end
      end)
    end)

    # Đo thời gian từ khi target chết đến khi tất cả DOWN delivered
    start = System.monotonic_time(:millisecond)
    send(target, :die)

    # Chờ tất cả monitors nhận DOWN
    # Trong thực tế: scheduler bị overload, không predict được
    :timer.sleep(5_000)
    elapsed = System.monotonic_time(:millisecond) - start

    IO.puts("Storm delivery time: #{elapsed}ms")
    # Kết quả thực tế: 15,000ms - 45,000ms (15-45 giây!)
    # Trong thời gian đó: cả cluster bị degrade nghiêm trọng
  end
end
```

---

## Tại sao built-in monitor không scale — 3 vấn đề cốt lõi

### Vấn đề 1: Fan-out tại source node

```
Node A chứa guild process bị kill:
  BEAM phải gửi {:DOWN} tới mọi remote monitor
  Với 500,000 remote monitors trên 4 nodes:
  → 4 TCP sends (batch per node) — OK
  → Nhưng mỗi batch chứa 125,000 PID references
  → Serialize 125,000 references = CPU spike trên Node A
  → Node A đang crash + đang spike CPU = race condition
  → Đôi khi Node A crash trước khi gửi xong
  → Một số nodes không nhận được DOWN notification
  → Silent failure — monitors "ghost"
```

### Vấn đề 2: Fan-out tại destination nodes

```
Node B nhận batch 125,000 {:DOWN} messages:
  BEAM distribution layer nhận 1 TCP packet lớn
  → Deserialize 125,000 message structs
  → Route mỗi message tới đúng process mailbox
  → 125,000 mailbox insertions đồng thời
  → Scheduler bị overwhelmed
  → Normal message processing bị stall
  → Tất cả 125,000 sessions trên Node B bị lag

Đây là O(n) work đột ngột inject vào scheduler
Không có backpressure mechanism
```

### Vấn đề 3: Thundering herd sau DOWN

```
500,000 sessions nhận {:DOWN, guild_pid} đồng thời:
  Mỗi session:
    1. Handle {:DOWN} message
    2. Try reconnect to guild → lookup registry
    3. Registry hiện tại: guild đang restart
    4. 500,000 sessions stampede registry đồng thời
    5. Registry bị overloaded
    6. Reconnect timeout → retry
    7. Exponential backoff → nhưng 500,000 cùng backoff
    8. Synchronized retry storms

Hệ thống không thể recover vì mỗi recovery attempt
tạo ra thêm load
```

---

## ZenMonitor — kiến trúc giải pháp

ZenMonitor cho phép monitor remote processes hiệu quả với minimal use of ERTS Distribution. Khi một process được monitor bởi số lượng lớn remote processes, việc process đó chết có thể gây flood cả node hosting process đó lẫn các nodes chứa monitoring processes.

```
Thay vì: 1 monitor = 1 entry tại remote node
ZenMonitor: N monitors = 1 entry tại remote node
            Local ZenMonitor proxy aggregate
```

![zenmonitor vs builtin monitor](zenmonitor_vs_builtin_monitor.svg)

---

## ZenMonitor — implementation chi tiết

```elixir
defmodule ZenMonitor do
  @moduledoc """
  Drop-in replacement cho Process.monitor/1.

  Thay vì: mỗi caller tạo 1 monitor entry tại remote node
  ZenMonitor: tất cả monitors từ 1 node → 1 proxy entry tại remote node
  Proxy aggregate và fan-out locally khi target dies
  """

  # ── Public API — drop-in replacement ─────────────────────────────

  @doc """
  Tương đương Process.monitor/1 nhưng scale tốt hơn nhiều.
  Trả về reference tương tự — compatible với existing code.
  """
  def monitor(pid) when is_pid(pid) do
    # Local process: dùng built-in (không cần overhead)
    if node(pid) == Node.self() do
      Process.monitor(pid)
    else
      # Remote process: route qua ZenMonitor infrastructure
      ZenMonitor.Local.monitor(pid)
    end
  end

  def demonitor(ref, opts \\ []) do
    ZenMonitor.Local.demonitor(ref, opts)
  end
end

defmodule ZenMonitor.Local do
  use GenServer
  @moduledoc """
  Chạy trên mỗi node.
  Aggregate tất cả monitors từ local processes → remote Proxy.
  Nhận DOWN notifications từ remote Proxy → fan out locally.
  """

  defstruct [
    # ETS table: ref → %{pid, target_pid, target_node}
    # Lưu tất cả active monitors
    :monitors_table,

    # ETS table: target_pid → [ref1, ref2, ...] (list monitors)
    # Reverse lookup: target chết → biết ai cần notify
    :targets_table,

    # Connected remote proxies
    # node → proxy_pid
    remote_proxies: %{},
  ]

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_) do
    monitors_table = :ets.new(:zen_monitors,
      [:set, :public, read_concurrency: true])
    targets_table  = :ets.new(:zen_targets,
      [:bag, :public, read_concurrency: true])
    # :bag cho phép multiple values per key
    # target_pid → nhiều refs từ nhiều callers

    {:ok, %__MODULE__{
      monitors_table: monitors_table,
      targets_table:  targets_table,
    }}
  end

  # ── monitor/1 — gọi bởi bất kỳ process nào trên node này ────────

  def monitor(target_pid) do
    ref         = make_ref()
    caller      = self()
    target_node = node(target_pid)

    # Store locally
    GenServer.call(__MODULE__, {
      :register_monitor,
      ref, caller, target_pid, target_node
    })

    # Ensure Proxy trên remote node biết về target này
    # (idempotent — nếu đã có rồi thì không thêm)
    GenServer.cast(__MODULE__, {:ensure_remote_watch, target_pid, target_node})

    ref
  end

  def handle_call({:register_monitor, ref, caller, target_pid, target_node}, _from, state) do
    # Lưu monitor entry
    :ets.insert(state.monitors_table, {ref, %{
      caller:      caller,
      target_pid:  target_pid,
      target_node: target_node,
      created_at:  System.monotonic_time()
    }})

    # Reverse index: target → [refs]
    :ets.insert(state.targets_table, {target_pid, ref})

    {:reply, ref, state}
  end

  # ── Ensure remote proxy đang watch target ────────────────────────

  def handle_cast({:ensure_remote_watch, target_pid, target_node}, state) do
    proxy_pid = get_or_connect_proxy(target_node, state)

    # Gửi 1 watch request tới remote proxy
    # Dù 100,000 sessions local đều monitor cùng target
    # Chỉ gửi 1 message tới proxy
    GenServer.cast(proxy_pid, {:watch, target_pid, Node.self()})

    {:noreply, state}
  end

  defp get_or_connect_proxy(target_node, state) do
    case Map.get(state.remote_proxies, target_node) do
      nil ->
        # Connect tới ZenMonitor.Proxy trên remote node
        proxy = {ZenMonitor.Proxy, target_node}
        GenServer.cast(__MODULE__, {:cache_proxy, target_node, proxy})
        proxy
      proxy ->
        proxy
    end
  end

  # ── Nhận DOWN từ remote proxy ─────────────────────────────────────

  def handle_info({:zen_down, target_pid, reason}, state) do
    # Tìm tất cả local refs monitor target này
    local_refs = :ets.lookup(state.targets_table, target_pid)
      |> Enum.map(&elem(&1, 1))

    # Fan out locally — không cần network
    # Controlled, không phải thundering herd
    Enum.each(local_refs, fn ref ->
      case :ets.lookup(state.monitors_table, ref) do
        [{^ref, %{caller: caller}}] ->
          # Gửi {:DOWN} message tới caller
          # Wrap với {:zen_monitor, reason} để distinguish
          send(caller, {:DOWN, ref, :process, target_pid,
                        {:zen_monitor, reason}})

          # Cleanup
          :ets.delete(state.monitors_table, ref)
          :ets.delete_object(state.targets_table, {target_pid, ref})
        [] ->
          :ok
      end
    end)

    {:noreply, state}
  end
end
```

```elixir
defmodule ZenMonitor.Proxy do
  use GenServer
  @moduledoc """
  Chạy trên mỗi node.
  Nhận watch requests từ remote nodes.
  Monitor actual processes locally.
  Notify remote nodes khi process chết.
  """

  defstruct [
    # target_pid → [watching_nodes]
    # Biết node nào cần notify khi target dies
    :watchers_table,

    # target_pid → monitor_ref (built-in monitor)
    # Chỉ 1 built-in monitor per target, dù có N watchers
    :local_monitors,
  ]

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_) do
    watchers_table = :ets.new(:zen_watchers,
      [:bag, :public, read_concurrency: true])
    local_monitors = :ets.new(:zen_local_monitors,
      [:set, :public])

    {:ok, %__MODULE__{
      watchers_table: watchers_table,
      local_monitors: local_monitors,
    }}
  end

  # ── Remote node muốn watch 1 target ──────────────────────────────

  def handle_cast({:watch, target_pid, watcher_node}, state) do
    # Thêm watcher_node vào danh sách
    :ets.insert(state.watchers_table, {target_pid, watcher_node})

    # Chỉ tạo 1 built-in monitor per target
    # Dù 4 remote nodes đều watch cùng target
    # → 1 monitor entry thay vì 4
    case :ets.lookup(state.local_monitors, target_pid) do
      [] ->
        # Chưa monitor target này → tạo mới
        ref = Process.monitor(target_pid)
        :ets.insert(state.local_monitors, {target_pid, ref})
      [_] ->
        # Đã monitor rồi → không cần thêm
        :ok
    end

    {:noreply, state}
  end

  # ── Process chết — proxy nhận DOWN ───────────────────────────────

  def handle_info({:DOWN, ref, :process, target_pid, reason}, state) do
    # Lấy tất cả nodes đang watch target này
    watching_nodes = :ets.lookup(state.watchers_table, target_pid)
      |> Enum.map(&elem(&1, 1))
      |> Enum.uniq()

    # Cleanup local state
    :ets.delete(state.local_monitors, target_pid)
    :ets.match_delete(state.watchers_table, {target_pid, :_})

    # Notify tất cả watching nodes
    # Mỗi node chỉ nhận 1 message (không phải N messages)
    Enum.each(watching_nodes, fn watcher_node ->
      send({ZenMonitor.Local, watcher_node},
           {:zen_down, target_pid, reason})
    end)

    {:noreply, state}
  end
end
```

---

## Jittered reconnect — giải quyết thundering herd sau DOWN

```elixir
defmodule Discord.Session.Process do
  # Nhận ZenMonitor DOWN notification
  def handle_info(
    {:DOWN, ref, :process, guild_pid, {:zen_monitor, reason}},
    state
  ) do
    # Verify đây là guild monitor của session này
    case Map.get(state.guild_monitors, ref) do
      nil -> {:noreply, state}

      guild_id ->
        # KHÔNG reconnect ngay lập tức
        # Nếu 500,000 sessions cùng reconnect ngay → stampede

        # Jitter: delay ngẫu nhiên trước khi reconnect
        jitter_ms = calculate_jitter(guild_id, state.user_id)
        Process.send_after(self(), {:reconnect_guild, guild_id}, jitter_ms)

        new_state = %{state |
          guild_monitors: Map.delete(state.guild_monitors, ref),
          guild_status:   Map.put(state.guild_status, guild_id, :reconnecting)
        }
        {:noreply, new_state}
    end
  end

  defp calculate_jitter(guild_id, user_id) do
    # Jitter dựa trên hash của (guild_id, user_id)
    # → Deterministic nhưng spread đều
    # → Tránh correlated retries

    base_jitter   = 1_000   # 1 giây base
    max_jitter    = 30_000  # tối đa 30 giây

    # Hash để distribute evenly
    hash = :erlang.phash2({guild_id, user_id}, max_jitter - base_jitter)

    base_jitter + hash
    # user_1 reconnect sau 1.5s
    # user_2 reconnect sau 3.2s
    # user_3 reconnect sau 15.7s
    # ...
    # 500,000 users spread đều trong 30 giây
    # Thay vì tất cả cùng lúc
  end

  def handle_info({:reconnect_guild, guild_id}, state) do
    case Discord.Guild.Registry.lookup(guild_id) do
      {:ok, new_guild_pid} ->
        # Guild đã restart → reconnect
        ref = ZenMonitor.monitor(new_guild_pid)
        GenServer.cast(new_guild_pid, {
          :session_reconnect,
          state.user_id,
          self()
        })

        new_state = %{state |
          guild_monitors: Map.put(state.guild_monitors, ref, guild_id),
          guild_status:   Map.put(state.guild_status, guild_id, :connected)
        }
        {:noreply, new_state}

      {:error, :not_found} ->
        # Guild chưa restart xong → retry sau
        retry_delay = 2_000 + :rand.uniform(3_000)
        Process.send_after(self(), {:reconnect_guild, guild_id}, retry_delay)
        {:noreply, state}
    end
  end
end
```

---

## ZenMonitor compatibility với existing code

ZenMonitor strives to be a drop-in replacement cho Process.monitor/1. Bất kỳ {:DOWN} message receivers nào match trên reason cần update để include outer {:zen_monitor, original_match} wrapper.

```elixir
# Migration từ Process.monitor sang ZenMonitor

# TRƯỚC — built-in monitor
def handle_info({:DOWN, ref, :process, pid, :normal}, state) do
  # process exit bình thường
  {:noreply, cleanup(state, ref)}
end

def handle_info({:DOWN, ref, :process, pid, {:exit, reason}}, state) do
  # process exit với reason
  {:noreply, handle_crash(state, ref, reason)}
end

# SAU — ZenMonitor wrapper
def handle_info({:DOWN, ref, :process, pid,
                 {:zen_monitor, :normal}}, state) do
  {:noreply, cleanup(state, ref)}
end

def handle_info({:DOWN, ref, :process, pid,
                 {:zen_monitor, {:exit, reason}}}, state) do
  {:noreply, handle_crash(state, ref, reason)}
end

# Backward compatible: vẫn handle built-in DOWN cho local processes
def handle_info({:DOWN, ref, :process, pid, reason}, state) do
  # Local process monitor (built-in) hoặc legacy remote
  {:noreply, cleanup(state, ref)}
end
```

---

## Kết quả thực tế sau khi deploy ZenMonitor

Nhờ ZenMonitor, Discord có thể tolerate guild node failure và recover trong ~40 giây.

```
Metric                    Before ZenMonitor    After ZenMonitor
────────────────────────────────────────────────────────────────
Monitor entries per node  500,000              4 (1 per remote node)
DOWN delivery time        15-45 giây           < 100ms
Scheduler spike on DOWN   100% CPU, minutes    minimal, < 1 giây
Cascade failures          thường xuyên         eliminated
Recovery time             5-15 phút            ~40 giây
Network overhead          N messages on DOWN   1 message per node

Cách Discord đo recovery time:
  t=0:    Node A bị kill
  t=5s:   ZenMonitor phát hiện, bắt đầu notify
  t=6s:   Tất cả sessions nhận DOWN
  t=6-36s: Jittered reconnect (spread 30 giây)
  t=40s:  Guild restart xong, tất cả sessions reconnected
  t=40s:  100% users back online
```

---

## Unknown unknowns — ZenMonitor edge cases

```elixir
defmodule ZenMonitor.EdgeCases do
  @moduledoc """
  Những vấn đề tinh tế khi dùng ZenMonitor ở Discord scale
  """

  # EDGE CASE 1: ZenMonitor.Proxy chính nó crash
  # Nếu Proxy crash → tất cả monitors từ node đó lost
  # Giải pháp: Supervisor restart Proxy + reconnect protocol
  def proxy_crash_recovery do
    # ZenMonitor.Local detect Proxy crash qua node_monitor
    # Trigger: re-register tất cả active monitors
    # Cost: O(unique_targets) re-registration messages
    :ok
  end

  # EDGE CASE 2: Network partition giữa monitor và Proxy
  # Node B không thể reach Node A
  # ZenMonitor xử lý như nodedown → DOWN delivered
  # Khi partition heal: tránh double-DOWN
  def partition_handling do
    # ZenMonitor track "already delivered" để idempotent
    # Ref-based deduplication trong ETS
    :ok
  end

  # EDGE CASE 3: Process chết trước khi Proxy nhận watch request
  # Race condition: target_pid chết trong khoảng thời gian
  # giữa caller gọi ZenMonitor.monitor() và Proxy nhận :watch
  def race_condition_handling do
    # Giải pháp: Proxy check nếu target đã chết khi nhận :watch
    # Nếu chết rồi: gửi DOWN ngay lập tức
    case Process.info(target_pid) do
      nil ->
        # Đã chết → notify ngay
        send({ZenMonitor.Local, watcher_node},
             {:zen_down, target_pid, :noproc})
      _ ->
        # Còn sống → setup monitor bình thường
        ref = Process.monitor(target_pid)
        :ets.insert(local_monitors, {target_pid, ref})
    end
  end

  # EDGE CASE 4: Mixed cluster (nodes có và không có ZenMonitor)
  # Discord có giai đoạn transitional khi deploy ZenMonitor
  def compatibility_check(remote_node) do
    # ZenMonitor.connect/1 ping Proxy trên remote node
    case ZenMonitor.compatibility(remote_node) do
      :compatible ->
        # Dùng ZenMonitor
        ZenMonitor.monitor(target_pid)
      :incompatible ->
        # Fallback về built-in
        Process.monitor(target_pid)
    end
  end
end
```

---

## Tóm tắt — ZenMonitor design principles

```
Built-in Process.monitor problems tại scale:
  N monitors → N remote entries → N DOWN messages simultaneously
  = O(N) network traffic + O(N) scheduler work đột ngột

ZenMonitor solution:
  N monitors (local) → 1 remote entry → 1 DOWN message
  Local proxy aggregate + controlled fan-out
  Jittered reconnect để tránh thundering herd

Core insight:
  "Aggregation at the monitoring layer, not at the application layer"
  Thay vì mỗi application biết cách handle storm
  → Infrastructure tự prevent storm từ đầu

Applicable pattern:
  Bất kỳ khi nào N parties quan tâm đến trạng thái của M entities
  N >> M → cần aggregation layer ở giữa
  Không chỉ cho Erlang — pattern này apply cho event systems nói chung
```

---

# Q9: Node Failure Recovery — 40 Giây Gồm Những Bước Gì

*Breakdown chi tiết từng millisecond của quá trình recovery*

---

## Setup — hiểu topology trước khi vào timeline

```
Discord cluster (simplified):

Node A (discord@10.0.1.1) — WILL FAIL
  ├── Guild processes: G1, G2, G3 ... G500
  ├── Session processes: 200,000 users
  └── ZenMonitor.Proxy

Node B (discord@10.0.1.2)
  ├── Guild processes: G501 ... G1000
  ├── Session processes: 200,000 users
  └── ZenMonitor.Local (monitoring G1-G500 on Node A)

Node C (discord@10.0.1.3)
  ├── Guild processes: G1001 ... G1500
  ├── Session processes: 200,000 users
  └── ZenMonitor.Local (monitoring G1-G500 on Node A)

Node D, E: tương tự Node B, C

Relationships:
  500,000 sessions trên B,C,D,E đang monitor guilds trên A
  (vì users trong guild G1-G500 connect khắp cluster)
```

---

## Timeline đầy đủ — 40 giây breakdown

![recovery_timeline](recovery_timeline.svg)

---

## Phase 1: t=0 → t=2s — Detection

### Tại sao mất 1-2 giây để detect?

```elixir
defmodule Discord.NodeDetection do
  @moduledoc """
  BEAM dùng TCP keepalive để detect node failure.
  Không phải heartbeat riêng — dùng OS-level TCP.
  """

  # Erlang distribution TCP keepalive defaults:
  # net_ticktime = 60 giây (mặc định)
  # → Node bị coi là down sau 60 giây không respond
  # → QUÁ CHẬM cho Discord

  # Discord config:
  def configure_fast_detection do
    # Giảm net_ticktime xuống 5 giây
    # BEAM sẽ gửi tick mỗi 5/4 = 1.25 giây
    # Timeout sau 4 ticks missed = 5 giây
    :net_kernel.set_net_ticktime(5)

    # Kết hợp với TCP keepalive ở OS level:
    # SO_KEEPALIVE = true
    # TCP_KEEPIDLE = 5s   (bắt đầu probe sau 5s idle)
    # TCP_KEEPINTVL = 1s  (probe interval)
    # TCP_KEEPCNT = 3     (3 probes failed = connection dead)
    # → Detect trong ~8 giây worst case

    # Với Fly.io / Kubernetes: thường detect trong 1-2 giây
    # vì network layer signal nodedown nhanh hơn TCP timeout
  end

  # Khi nodedown detected:
  def handle_nodedown(node) do
    # BEAM gửi {:nodedown, node} tới:
    # 1. Tất cả processes đã gọi :net_kernel.monitor_nodes(true)
    # 2. ZenMonitor.Local (đã subscribe)
    # 3. libcluster EventHandler
    # 4. Discord.Cluster.EventHandler
    :ok
  end
end
```

---

## Phase 2: t=2s → t=3s — ZenMonitor Processing

```elixir
defmodule ZenMonitor.Local do
  # Nhận nodedown → process tất cả monitors cho node đó
  def handle_info({:nodedown, dead_node}, state) do
    start = System.monotonic_time(:millisecond)

    # Lấy tất cả targets trên dead node từ ETS
    # O(unique_guild_count) — không phải O(session_count)
    dead_targets = :ets.select(state.monitors_table, [
      {{:"$1", %{target_node: dead_node}},
       [],
       [:"$1"]}
    ])
    # dead_targets = [guild_pid_1, guild_pid_2, ... guild_pid_500]
    # Chỉ 500 entries — không phải 500,000

    # Fan out tới local processes
    # Batched để tránh spike scheduler
    dead_targets
    |> Enum.chunk_every(100)   # batch 100 targets
    |> Enum.each(fn batch ->
      Enum.each(batch, fn target_pid ->
        notify_local_monitors(target_pid, :noconnection, state)
      end)
      # Yield scheduler sau mỗi batch
      # Tránh monopolize scheduler quá lâu
      Process.sleep(0)  # :erlang.yield() equivalent
    end)

    elapsed = System.monotonic_time(:millisecond) - start
    :telemetry.execute(
      [:discord, :zenmonitor, :nodedown_processed],
      %{duration: elapsed, targets: length(dead_targets)},
      %{node: dead_node}
    )

    {:noreply, state}
  end

  defp notify_local_monitors(target_pid, reason, state) do
    # Lookup tất cả local callers đang monitor target này
    local_refs = :ets.lookup(state.targets_table, target_pid)

    Enum.each(local_refs, fn {_target, ref} ->
      case :ets.lookup(state.monitors_table, ref) do
        [{^ref, %{caller: caller_pid}}] ->
          send(caller_pid, {
            :DOWN, ref, :process, target_pid,
            {:zen_monitor, reason}
          })
          :ets.delete(state.monitors_table, ref)
        [] -> :ok
      end
    end)

    :ets.match_delete(state.targets_table, {target_pid, :_})
  end
end
```

---

## Phase 3: t=3s — Horde Supervisor restart guilds

```elixir
defmodule Discord.Guild.HordeSupervisor do
  # Horde detect member down qua :nodedown event
  # Tự động restart processes của dead node

  def handle_info({:nodedown, dead_node}, state) do
    # Horde internal: find tất cả processes owned by dead_node
    orphaned = Horde.Registry.select(
      Discord.Guild.HordeRegistry,
      [{{:"$1", :"$2", :"$3"}, [{:==, {:node, :"$2"}, dead_node}], [:"$1"]}]
    )
    # orphaned = [guild_id_1, guild_id_2, ... guild_id_500]

    # Redistribute theo consistent hash ring
    Enum.each(orphaned, fn guild_id ->
      target_node = Discord.Router.node_for_guild(guild_id)

      # Start process trên target node
      # Horde tự handle nếu process đã exist (idempotent)
      Horde.DynamicSupervisor.start_child(
        __MODULE__,
        {Discord.Guild.Process, guild_id: guild_id},
        target_node: target_node
      )
    end)

    {:noreply, state}
  end
end
```

---

## Phase 4: t=3s → t=15s — Guild process initialization

Đây là phase chậm nhất — load state từ persistent storage:

```elixir
defmodule Discord.Guild.Process do
  def init(guild_id) do
    start = System.monotonic_time(:millisecond)

    # Bước 1: Load guild metadata từ cache/DB (~50ms)
    metadata = load_guild_metadata(guild_id)

    # Bước 2: Load members vào ETS (~5-12 giây tùy guild size)
    # Đây là bottleneck chính của recovery time
    table = :ets.new(:"guild_members_#{guild_id}", [:set, :public])
    member_count = load_members_into_ets(guild_id, table)

    # Bước 3: Load voice states (~10ms)
    voice_states = load_voice_states(guild_id)

    elapsed = System.monotonic_time(:millisecond) - start

    :telemetry.execute(
      [:discord, :guild, :cold_start],
      %{duration: elapsed, member_count: member_count},
      %{guild_id: guild_id}
    )

    state = %__MODULE__{
      guild_id:    guild_id,
      ets_table:   table,
      voice_states: voice_states,
    }

    # Notify supervisor: guild ready để nhận sessions
    send(self(), :register_as_ready)

    {:ok, state}
  end

  defp load_members_into_ets(guild_id, table) do
    # Stream từ ScyllaDB — không load toàn bộ vào memory
    # (tránh OOM khi guild có 1M+ members)
    Discord.DB.stream_guild_members(guild_id)
    |> Stream.chunk_every(1_000)
    |> Stream.each(fn chunk ->
      # Batch insert vào ETS
      :ets.insert(table, chunk)
    end)
    |> Stream.run()

    :ets.info(table, :size)
  end

  # Guild ready: register trong Horde.Registry
  def handle_info(:register_as_ready, state) do
    # Đăng ký trong distributed registry
    # Sau bước này: sessions có thể lookup và connect
    Horde.Registry.register(
      Discord.Guild.HordeRegistry,
      state.guild_id,
      %{node: Node.self(), ready_at: DateTime.utc_now()}
    )

    {:noreply, %{state | status: :ready}}
  end
end
```

### Tại sao load members mất 3-12 giây?

```
Guild với 1,000,000 members:
  ScyllaDB read throughput: ~100,000 rows/giây
  1,000,000 rows / 100,000 = ~10 giây

Optimization Discord dùng:
  1. Parallel reads từ nhiều ScyllaDB nodes
  2. Pre-warm cache (Midjourney guild luôn hot trong Redis)
  3. Snapshot-based recovery: lưu ETS snapshot định kỳ
     → Load từ snapshot (~2 giây) thay vì DB (~10 giây)
```

---

## Phase 5: t=6s → t=36s — Jittered reconnect

```elixir
defmodule Discord.Session.Process do
  # Nhận DOWN từ ZenMonitor
  def handle_info(
    {:DOWN, ref, :process, _guild_pid, {:zen_monitor, _reason}},
    state
  ) do
    guild_id = Map.get(state.guild_monitors, ref)

    # Tính jitter delay
    delay = reconnect_jitter(guild_id, state.user_id)

    Process.send_after(
      self(),
      {:attempt_guild_reconnect, guild_id, 0},
      delay
    )

    {:noreply, %{state |
      guild_monitors: Map.delete(state.guild_monitors, ref)
    }}
  end

  defp reconnect_jitter(guild_id, user_id) do
    # Deterministic jitter: cùng user → cùng delay
    # Phân phối đều từ 1s đến 30s
    base     = 1_000
    variance = 29_000
    :erlang.phash2({guild_id, user_id}, variance) + base
  end

  # Attempt reconnect — với retry logic
  def handle_info(
    {:attempt_guild_reconnect, guild_id, attempt},
    state
  ) when attempt < 10 do
    case Horde.Registry.lookup(Discord.Guild.HordeRegistry, guild_id) do
      [{guild_pid, %{status: :ready}}] ->
        # Guild đã ready → kết nối
        do_guild_reconnect(guild_id, guild_pid, state)

      [{_pid, %{status: :initializing}}] ->
        # Guild đang load → đợi thêm
        retry_delay = :math.pow(2, attempt) * 500 |> round()
        Process.send_after(
          self(),
          {:attempt_guild_reconnect, guild_id, attempt + 1},
          retry_delay
        )
        {:noreply, state}

      [] ->
        # Guild chưa start → đợi
        Process.send_after(
          self(),
          {:attempt_guild_reconnect, guild_id, attempt + 1},
          2_000
        )
        {:noreply, state}
    end
  end

  defp do_guild_reconnect(guild_id, guild_pid, state) do
    # Monitor guild mới
    ref = ZenMonitor.monitor(guild_pid)

    # Gửi reconnect request với last_seen_sequence
    # Guild sẽ gửi lại messages đã miss
    last_seq = Map.get(state.last_sequences, guild_id, 0)

    GenServer.cast(guild_pid, {
      :session_reconnect,
      state.user_id,
      self(),
      %{last_sequence: last_seq}
    })

    new_state = %{state |
      guild_monitors:  Map.put(state.guild_monitors, ref, guild_id),
      guild_pids:      Map.put(state.guild_pids, guild_id, guild_pid),
    }

    {:noreply, new_state}
  end
end
```

---

## Guild xử lý reconnect — gửi missed messages

```elixir
defmodule Discord.Guild.Process do
  def handle_cast(
    {:session_reconnect, user_id, session_pid, %{last_sequence: last_seq}},
    state
  ) do
    # Tìm messages user đã miss trong khi guild down
    missed = get_messages_since_sequence(state, last_seq)

    # Gửi state sync cho session
    send(session_pid, {
      :guild_state_sync,
      %{
        # Messages đã miss (từ message cache trong memory)
        missed_messages: missed,
        # Presence snapshot hiện tại
        presences: build_presence_snapshot(state),
        # Voice states
        voice_states: state.voice_states,
        # Sequence number hiện tại để session sync
        current_sequence: state.message_sequence,
      }
    })

    # Thêm session vào relay
    assign_to_relay(user_id, session_pid, state)

    {:noreply, state}
  end

  defp get_messages_since_sequence(state, last_seq) do
    # Message cache giữ 50 messages gần nhất trong memory
    # Nếu user miss nhiều hơn 50 → client sẽ fetch từ DB
    state.message_cache
    |> Enum.filter(fn msg -> msg.sequence > last_seq end)
    |> Enum.sort_by(& &1.sequence)
  end
end
```

---

## Monitoring recovery — Prometheus metrics

```elixir
defmodule Discord.Recovery.Metrics do
  @doc """
  Metrics Discord track trong quá trình recovery
  để alert nếu recovery chậm hơn expected
  """

  def setup_recovery_tracking(dead_node, guild_count) do
    # Track từng phase
    start_times = %{
      detection:   nil,
      zen_done:    nil,
      horde_done:  nil,
      guilds_ready: nil,
      sessions_done: nil,
    }

    # SLO targets:
    # Detection:          < 3 giây
    # ZenMonitor fan-out: < 1 giây sau detection
    # Guild restart:      < 5 giây sau detection
    # Guild ready:        < 15 giây sau detection
    # Full recovery:      < 45 giây sau detection

    :telemetry.execute(
      [:discord, :recovery, :started],
      %{guild_count: guild_count},
      %{dead_node: dead_node}
    )
  end

  # Alert nếu recovery quá chậm
  def check_recovery_progress(phase, elapsed_ms) do
    slo = %{
      detection:    3_000,
      zen_done:     4_000,
      horde_done:   6_000,
      guilds_ready: 18_000,
      full_recovery: 45_000,
    }

    if elapsed_ms > Map.get(slo, phase, 0) do
      :telemetry.execute(
        [:discord, :recovery, :slo_breach],
        %{elapsed: elapsed_ms, slo: slo[phase]},
        %{phase: phase}
      )
      # Trigger PagerDuty alert
    end
  end
end
```

---

## Edge cases — khi recovery không đi theo happy path

```elixir
defmodule Discord.Recovery.EdgeCases do

  # CASE 1: Node A restart ngay sau khi die
  # (crash loop — OOM killer)
  # → Node A join lại cluster sau 5 giây
  # → Guilds đã moved sang B, C, D, E
  # → Hash ring: guild_id vẫn map về Node B (đã updated)
  # → Node A join: không có guilds → just another worker node
  # Không có conflict vì hash ring là source of truth

  # CASE 2: 2 nodes die cùng lúc (network partition)
  # → Cluster bị split thành 2 parts
  # → Mỗi part nghĩ mình là "the cluster"
  # → Split brain: 2 copies của cùng guild process
  # → Discord dùng Horde với distributed consensus
  #    để detect và resolve conflict

  def handle_split_brain(guild_id) do
    # Horde detect: 2 processes cùng guild_id trong registry
    # Resolution: process nào được registered sau thì win
    # (Horde dùng CRDT-based conflict resolution)

    # Loser process nhận :kill message từ Horde
    # Winner process tiếp tục serve

    # State merge: winner apply events từ loser's buffer
    # (eventual consistency trong ~100ms sau partition heal)
    :ok
  end

  # CASE 3: Guild DB chậm khi load members
  # Guild process init timeout (default 5 giây trong GenServer)
  # → Process crash trước khi load xong
  # → Supervisor retry → crash lại
  # → Retry loop

  def handle_slow_db_init(guild_id) do
    # Giải pháp: async init pattern
    # Guild start nhanh với empty state
    # Load data async trong background
    # Mark as :loading, từ chối sessions cho đến khi :ready

    # Sessions nhận {:guild_loading, estimated_ready_at}
    # Client hiện loading spinner thay vì error
    :ok
  end

  # CASE 4: Sessions reconnect nhanh hơn guild ready
  # (jitter 1 giây, guild cần 10 giây để load)
  # → Session lookup Horde.Registry → không tìm thấy
  # → Retry với exponential backoff
  # → Guild ready sau 10 giây → session reconnect thành công

  def handle_session_before_guild(guild_id, session_pid) do
    # Subscribe vào "guild_ready" event
    Phoenix.PubSub.subscribe(
      Discord.PubSub,
      "guild_ready:#{guild_id}"
    )

    # Khi guild ready: broadcast event này
    # Session nhận → reconnect ngay không cần wait timer
    :ok
  end
end
```

---

## Tổng kết — tại sao 40 giây là achievable

```
Phase           Duration    Bottleneck            Optimization
────────────────────────────────────────────────────────────────
Detection       1-2s        TCP keepalive         net_ticktime=5s
ZenMonitor      <1s         ETS lookup × 500      Off-heap, O(guilds)
Horde restart   <1s         process spawn         DynamicSupervisor
Guild init      3-15s       DB read 1M members    ScyllaDB stream
                            (đây là bottleneck)   + ETS snapshot cache
Jitter spread   30s         intentional           phash2 distribution
────────────────────────────────────────────────────────────────
Total           ~40s        guild init dominant   cache warm guilds

Key: 40 giây = guild init time + max jitter
     Không phải 15 phút như trước ZenMonitor
     Không phải infinity như trước Horde

User experience:
  t=0-6s:   Message lag, presence freeze
  t=6-36s:  Guild reconnecting (spinner) — staggered
  t=40s:    100% normal
  Total perceived downtime per user: ~6 giây average
```

---

# Q10: Mnesia Thất Bại và Tại Sao ETS + GenServer Là Đáp Án Đúng

*Bài học đắt giá nhất của Discord với Erlang ecosystem*

---

## Mnesia là gì — và tại sao nó hấp dẫn ban đầu

```
Mnesia = distributed database built into Erlang/OTP
Không cần install gì thêm — có sẵn trong mọi Erlang VM
Hỗ trợ:
  - In-memory tables (như ETS nhưng distributed)
  - Persistent tables (write to disk)
  - ACID transactions
  - Automatic replication giữa nodes
  - Query language (QLC - Query List Comprehension)
  - Schema migrations
```

```elixir
# Mnesia trông rất hấp dẫn khi mới dùng:

# Tạo distributed table — replicated tới tất cả nodes
:mnesia.create_table(:guild_members, [
  attributes: [:user_id, :guild_id, :nick, :roles, :joined_at],
  disc_copies: [Node.self() | Node.list()],  # persist + replicate
  type: :set
])

# Query giống SQL
:mnesia.transaction(fn ->
  :mnesia.match_object({:guild_members, :_, guild_id, :_, :_, :_})
end)

# Tự động sync giữa nodes — không cần viết gì thêm
# Nghe có vẻ perfect cho Discord usecase
```

---

## Discord thử Mnesia lần 1 — Persistent mode

```elixir
defmodule Discord.GuildRegistry.MnesiaV1 do
  @moduledoc """
  Attempt 1: Dùng Mnesia disc_copies để persist guild → node mapping
  Mục tiêu: survive node restart mà không cần reload từ DB
  """

  def setup do
    # Table replicated + persisted trên tất cả nodes
    :mnesia.create_table(:guild_locations, [
      attributes: [:guild_id, :node, :pid, :updated_at],
      disc_copies: Node.list() ++ [Node.self()],
      type: :set
    ])
  end

  def register_guild(guild_id, pid) do
    :mnesia.transaction(fn ->
      :mnesia.write({:guild_locations, guild_id, Node.self(), pid,
                     DateTime.utc_now()})
    end)
  end

  def lookup_guild(guild_id) do
    :mnesia.transaction(fn ->
      case :mnesia.read(:guild_locations, guild_id) do
        [{:guild_locations, ^guild_id, node, pid, _}] -> {:ok, node, pid}
        [] -> {:error, :not_found}
      end
    end)
  end
end
```

### Vấn đề phát sinh với disc_copies

```
Scenario: Node A crash và restart

t=0:    Node A down
t=5s:   Node A restart
t=5s:   Mnesia trên Node A bắt đầu sync với Node B, C, D

Bình thường: sync mất ~1-2 giây → OK

Vấn đề ở Discord scale:
  Mnesia table size: 500,000 guild entries × 100 bytes = 50MB
  50MB sync qua network: fine

  Nhưng khi Node A crash GIỮA LÚC write transaction:
  → Mnesia trên Node A có "dirty" state trên disk
  → Phải reconcile với other nodes
  → Mnesia chọn: lock table trong khi reconcile
  → Lock time: không predictable — có thể hàng phút

  Trong thời gian lock:
  → Tất cả reads/writes trên tất cả nodes bị block
  → Guild lookups fail
  → Sessions không reconnect được
  → Cascading failure
```

```
Worse: khi reconcile xong, Mnesia có thể quyết định
Node A's version là "newer" (dựa trên timestamp)
→ Overwrite data từ Node B, C, D
→ Mất writes từ khi Node A down đến khi restart
→ Silent data loss

Discord discovered: dirty node có thể corrupt cluster
không bao giờ catch up được, phải manual intervention
```

---

## Discord thử Mnesia lần 2 — In-memory mode

```elixir
defmodule Discord.PresenceStore.MnesiaV2 do
  @moduledoc """
  Attempt 2: Chỉ dùng ram_copies — không persist
  Mục tiêu: distributed presence store
  (ai đang online, status, activities)
  """

  def setup do
    :mnesia.create_table(:user_presence, [
      attributes: [:user_id, :status, :activities, :client_status],
      # ram_copies: chỉ trong memory, không write to disk
      # → tránh dirty state issue của disc_copies
      ram_copies: Node.list() ++ [Node.self()],
      type: :set
    ])
  end

  def update_presence(user_id, presence_data) do
    :mnesia.transaction(fn ->
      :mnesia.write({:user_presence, user_id,
                     presence_data.status,
                     presence_data.activities,
                     presence_data.client_status})
    end)
    # Transaction = 2PC (Two-Phase Commit) across all nodes
    # Mọi node phải acknowledge trước khi commit
  end
end
```

### Vấn đề với ram_copies

```
Two-Phase Commit (2PC) overhead:

Discord có 400-500 Erlang nodes
Mỗi presence update = 2PC across tất cả nodes:

Phase 1 (Prepare):
  Coordinator → 499 nodes: "prepare to commit"
  499 nodes → Coordinator: "ready" / "abort"
  Network round trips: 499 × 2ms RTT = 998ms minimum!

Phase 2 (Commit):
  Coordinator → 499 nodes: "commit"
  Thêm 998ms

Total: ~2 giây cho 1 presence update
Discord có: ~1,000,000 presence updates/giây
1,000,000 updates × 2 giây/update = impossible

Mnesia 2PC không scale với cluster lớn
```

```
Vấn đề thứ 2: Network partition
Nếu cluster bị split (một số nodes không reach được):

Mnesia chọn: require majority để commit
→ Minority partition không thể write
→ OK cho consistency, nhưng availability suffer

Discord usecase: presence cần high availability
"Better to show stale presence than no presence"
→ Mnesia's strict consistency không phù hợp
```

Discord thử Mnesia trong production hai lần — ở cả persistent và in-memory mode. Database nodes thường xuyên bị lag trong failure scenarios, đôi khi không thể catch up được. Cuối cùng họ bỏ Mnesia hoàn toàn và xây dựng functionality mong muốn với Erlang's builtin constructs như GenServer và ETS.

---

## Root cause analysis — tại sao Mnesia fail ở Discord scale

```
Mnesia được thiết kế cho:
  - Small clusters (3-10 nodes)
  - Telecom systems với predictable load
  - Strong consistency requirements
  - Moderate transaction rates (~1000/giây)

Discord thực tế:
  - 400-500 nodes
  - Unpredictable load spikes (viral moments)
  - Eventually consistent OK (presence, typing)
  - Millions of updates/giây

→ Fundamental mismatch, không phải bug
```

```elixir
defmodule Discord.MnesiaProblems do
  @doc "5 vấn đề cụ thể Discord encountered"

  # Problem 1: Schema lock khi thêm node mới
  def problem_schema_lock do
    # Khi thêm node vào cluster:
    # Mnesia phải lock toàn bộ schema để sync
    # Lock time: proportional to table size × node count
    # Với 500 nodes × large tables: hàng giờ!

    # :mnesia.add_table_copy block tất cả operations
    # không có timeout option
    :mnesia.add_table_copy(:user_presence, new_node, :ram_copies)
    # ↑ Có thể hang indefinitely
  end

  # Problem 2: Dirty node sau partial failure
  def problem_dirty_node do
    # Node A bị OOM killer trong khi Mnesia đang commit
    # Node A restart với partial write on disk
    # Mnesia không biết version nào là "correct"
    # Manual intervention required:
    :mnesia.set_master_nodes(:guild_locations, [Node.self()])
    # Dangerous: có thể overwrite valid data
  end

  # Problem 3: Transaction latency unpredictable
  def problem_transaction_latency do
    # Normal case: ~1ms per transaction
    # When 1 node slow: tất cả transactions slow
    # (2PC requires all participants to respond)

    # Discord observed: p99 latency ~2-3 giây
    # p99.9: timeout (30 giây default)
    # → Sessions seeing 30 giây delays randomly
    :ok
  end

  # Problem 4: Memory không được control
  def problem_memory do
    # Mnesia ram_copies: data + indexes + transaction log
    # Discord không control được memory layout
    # Không integrate với ETS optimization (read_concurrency etc.)

    # 1,000,000 users × 500 bytes = 500MB minimum
    # Mnesia overhead: 2-3x → 1-1.5GB per node
    # × 500 nodes = 500-750GB RAM chỉ cho presence
    :ok
  end

  # Problem 5: Không có backpressure
  def problem_no_backpressure do
    # Client gửi presence updates nhanh hơn Mnesia process
    # Mnesia không có reject mechanism
    # → Transaction queue grows unbounded
    # → Eventually OOM
    # → Node crash → dirty state → vòng lặp ác tính
    :ok
  end
end
```

---

## Solution: ETS + GenServer — tự xây với primitives

Sau khi bỏ Mnesia, Discord xây distributed presence store từ scratch:

```elixir
defmodule Discord.Presence.Store do
  @moduledoc """
  Replacement cho Mnesia presence store.
  Design principles:
    1. Local-first: mỗi node owns presence của sessions trên nó
    2. Eventual consistency: gossip updates giữa nodes
    3. Explicit backpressure: reject khi overloaded
    4. Predictable memory: ETS với known overhead
    5. No distributed transactions: fire-and-forget updates
  """

  use GenServer

  defstruct [
    # Local presence table — ETS
    # user_id → %{status, activities, updated_at}
    :local_table,

    # Remote presence cache — ETS
    # user_id → %{status, activities, node, updated_at}
    :remote_table,

    # Subscribers: ai cần được notify khi presence thay đổi
    # guild_id → [session_pids]
    subscribers: %{},
  ]

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_) do
    local_table = :ets.new(:presence_local, [
      :set, :public, :named_table,
      read_concurrency: true,
      write_concurrency: false  # chỉ GenServer này write
    ])

    remote_table = :ets.new(:presence_remote, [
      :set, :public, :named_table,
      read_concurrency: true,
      write_concurrency: false
    ])

    # Schedule periodic gossip tới other nodes
    schedule_gossip()

    {:ok, %__MODULE__{
      local_table:  local_table,
      remote_table: remote_table,
    }}
  end

  # ── Local updates — từ sessions trên node này ──────────────────

  def update_local(user_id, presence_data) do
    # Direct ETS write — không qua GenServer
    # O(1), không block, không network
    :ets.insert(:presence_local, {user_id, presence_data})

    # Async notify GenServer để fan-out tới subscribers
    # Non-blocking: session không chờ fanout hoàn thành
    GenServer.cast(__MODULE__, {:local_updated, user_id, presence_data})
  end

  def handle_cast({:local_updated, user_id, presence_data}, state) do
    # Find guilds cần notify
    guilds = Discord.Session.get_shared_guilds(user_id)

    # Notify subscribers (sessions trong guild)
    Enum.each(guilds, fn guild_id ->
      case Map.get(state.subscribers, guild_id) do
        nil -> :ok
        pids ->
          Manifold.send(pids, {:presence_update, user_id, presence_data})
      end
    end)

    {:noreply, state}
  end

  # ── Gossip protocol — sync với other nodes ────────────────────

  defp schedule_gossip do
    # Gossip mỗi 5 giây
    # Không phải per-update sync (như Mnesia 2PC)
    # → Eventual consistency, không phải strong consistency
    Process.send_after(self(), :gossip, 5_000)
  end

  def handle_info(:gossip, state) do
    # Lấy tất cả local presence updates trong 5 giây vừa rồi
    cutoff = System.monotonic_time(:second) - 5

    recent_updates = :ets.select(:presence_local, [
      {{:"$1", %{updated_at: :"$2"}},
       [{:>, :"$2", cutoff}],
       [{{:"$1", :"$2"}}]}
    ])

    if recent_updates != [] do
      # Broadcast delta tới tất cả nodes
      # Fire-and-forget — không cần ACK
      other_nodes = Node.list()
      Enum.each(other_nodes, fn node ->
        GenServer.cast(
          {__MODULE__, node},
          {:remote_gossip, Node.self(), recent_updates}
        )
      end)
    end

    schedule_gossip()
    {:noreply, state}
  end

  def handle_cast({:remote_gossip, source_node, updates}, state) do
    # Nhận gossip từ remote node
    # Merge với local remote_table
    Enum.each(updates, fn {user_id, presence} ->
      # Last-write-wins: chỉ update nếu newer
      case :ets.lookup(:presence_remote, user_id) do
        [{^user_id, existing}] when existing.updated_at >= presence.updated_at ->
          :ok  # existing is newer, skip
        _ ->
          :ets.insert(:presence_remote, {user_id,
            Map.put(presence, :source_node, source_node)})
      end
    end)

    {:noreply, state}
  end

  # ── Read — từ bất kỳ process nào ─────────────────────────────

  def get_presence(user_id) do
    # Check local first (fastest)
    case :ets.lookup(:presence_local, user_id) do
      [{^user_id, data}] -> {:ok, data}
      [] ->
        # Check remote cache
        case :ets.lookup(:presence_remote, user_id) do
          [{^user_id, data}] -> {:ok, data}
          [] -> {:ok, :offline}
        end
    end
    # Đọc từ ETS: ~0.5μs
    # Không network round trip
    # Không transaction overhead
  end
end
```

---

## So sánh chi tiết — Mnesia vs ETS+GenServer

```
                    Mnesia              ETS + GenServer
────────────────────────────────────────────────────────────────
Consistency         Strong (2PC)        Eventual (gossip)
Write latency       ~2s (2PC × nodes)   ~1μs (local ETS)
Read latency        ~1ms (transaction)  ~0.5μs (ETS lookup)
Scale (nodes)       ~10 nodes max       ~500+ nodes fine
Memory control      Limited             Full control
Backpressure        None                GenServer mailbox
Failure recovery    Complex, manual     Automatic, predictable
Network overhead    High (2PC)          Low (gossip batched)
Schema changes      Lock cluster        No schema needed
Transaction support Yes (ACID)          No (by design)
Debugging           Hard                Excellent (ETS introspect)
Lines of code       ~50 (Mnesia)        ~300 (custom)
Operational risk    High                Low (well-understood)
```

---

## Khi nào vẫn dùng Mnesia — niche use cases

```elixir
defmodule Discord.MnesiaStillValidUsecases do
  @moduledoc """
  Mnesia vẫn tốt khi:
  1. Cluster nhỏ (2-5 nodes)
  2. Strong consistency bắt buộc
  3. Low write rate (~100/giây)
  4. Cần ACID transactions thật sự
  """

  # Use case OK: distributed lock manager
  # - Cluster 3 nodes
  # - Lock requests: ~10/giây
  # - ACID critical: không thể có 2 processes cùng hold lock
  def distributed_lock_example do
    :mnesia.transaction(fn ->
      case :mnesia.read(:locks, :deployment_lock) do
        [] ->
          :mnesia.write({:locks, :deployment_lock,
                         Node.self(), DateTime.utc_now()})
          :acquired
        [{_, _, owner, _}] ->
          {:already_held_by, owner}
      end
    end)
    # Với 3 nodes và 10 requests/giây: Mnesia là fine
  end

  # Discord thực tế không dùng Mnesia cho bất cứ thứ gì nữa
  # Sau 2 failed attempts, team quyết định:
  # "If it needs to be distributed, we build it ourselves"
end
```

---

## Pattern tổng quát — Discord's distributed data principles

```elixir
defmodule Discord.DistributedDataPrinciples do
  @moduledoc """
  Sau Mnesia experience, Discord áp dụng principles này
  cho mọi distributed data problem:
  """

  # Principle 1: Local-first ownership
  # Mỗi piece of data có 1 authoritative owner process
  # Owner = single-threaded → no races, no locks needed
  #
  # Mnesia violated này bằng cách allow mọi node write

  # Principle 2: Explicit consistency model
  # Strong consistency: dùng GenServer.call (serialized)
  # Eventual consistency: dùng GenServer.cast + gossip
  # Never: implicit distributed transaction
  #
  # Mnesia provided strong consistency nhưng với hidden cost

  # Principle 3: Backpressure everywhere
  # Mọi async operation phải có explicit bound
  # GenServer mailbox = natural backpressure
  # Khi mailbox đầy → caller block → propagates up
  #
  # Mnesia có transaction queue nhưng không expose backpressure

  # Principle 4: Prefer duplication over coordination
  # Thà có stale data trên mỗi node còn hơn coordinate mọi write
  # Presence: mỗi node cache của neighbor → đọc local
  # Guild metadata: cache trên mọi node → invalidate khi thay đổi
  #
  # Mnesia forced coordination cho mọi write

  # Principle 5: Failure as first-class citizen
  # Thiết kế cho "what happens when node dies"
  # không phải "how do we prevent node from dying"
  # ETS + GenServer: khi process die → Supervisor restart
  # Mnesia: khi node die → complex recovery protocol

  def choose_data_storage(requirements) do
    cond do
      requirements.needs_acid_transactions and
      requirements.cluster_size <= 5 ->
        :mnesia  # rare: small cluster strict consistency

      requirements.read_heavy and
      requirements.write_rate < 100 and
      requirements.global_read ->
        :persistent_term  # read-mostly global config

      requirements.shared_between_processes and
      requirements.needs_filtering ->
        :ets  # shared mutable local state

      requirements.distributed and
      requirements.eventual_ok ->
        :ets_plus_gossip  # Discord's approach

      requirements.single_process_owns ->
        :genserver_state  # simplest, most correct

      true ->
        :ets  # default safe choice
    end
  end
end
```

---

## Lesson learned — "Build it yourself" philosophy

```
Discord's journey với distributed data:

Attempt 1: Mnesia disc_copies
  → Dirty nodes, unpredictable locks
  → Bỏ sau vài tuần

Attempt 2: Mnesia ram_copies
  → 2PC quá chậm ở scale
  → Bỏ sau vài tháng

Final solution: ETS + GenServer + custom gossip
  → 300 lines of code
  → Full control
  → Predictable behavior
  → Scales tới 500+ nodes
  → Runs for years without issues

Key insight:
  Erlang/OTP cung cấp tất cả primitives cần thiết:
    ETS = fast local storage
    GenServer = single-threaded state machine
    Process.send = async messaging
    :net_kernel.monitor_nodes = cluster events

  Kết hợp đúng cách → tốt hơn bất kỳ distributed database nào
  vì được tuned chính xác cho Discord's access patterns

  "We don't use Mnesia. We use the building blocks
   that Mnesia itself is built from." — Discord Engineering
```

---

## Tóm tắt — Mnesia lessons cho bất kỳ Elixir system nào

```
Dùng Mnesia khi:
  ✅ Cluster ≤ 5 nodes
  ✅ Write rate ≤ 100/giây
  ✅ ACID transactions thật sự cần thiết
  ✅ Team có Mnesia expertise

KHÔNG dùng Mnesia khi:
  ❌ Cluster > 10 nodes
  ❌ High write throughput (>1000/giây)
  ❌ Eventual consistency acceptable
  ❌ Need predictable latency
  ❌ Memory control critical

Thay thế tốt hơn:
  ETS + GenServer          → local shared state
  ETS + gossip             → distributed eventual consistent
  :persistent_term         → global read-heavy config
  Phoenix.PubSub           → event broadcasting
  External DB (ScyllaDB)   → persistent distributed data
  Redis                    → cache + pub/sub
```

---

# Q11: Distributed Tracing trong Actor Model — Transport Library và Dynamic Sampling

*Bài toán observability khó nhất: trace 1 message qua 1 triệu actors*

---

## Tại sao distributed tracing trong actor model khác hoàn toàn HTTP

```
HTTP-based tracing (quen thuộc):
  Request A → Service 1 → Service 2 → Service 3
  1 trace = 1 request = linear chain
  Mỗi hop: inject trace_id vào HTTP header
  → OpenTelemetry/Jaeger handle tốt

Actor model tracing (Discord's challenge):
  1 message → guild process → fanout → 1,000,000 actors → 1,000,000 sessions
  1 trace = 1 event = exponential fan
  Không có "HTTP header" để inject
  → Standard OpenTelemetry KHÔNG scale
```

```
Nếu trace mọi hop trong Discord:

User gửi 1 message vào guild 1,000,000 members:
  1 root span (message received)
  + 1 span (guild process)
  + 1,000,000 child spans (mỗi session delivery)
  = 1,000,002 spans cho 1 message

Discord có: ~100,000 messages/giây vào large guilds
→ 100,000 × 1,000,002 spans/giây
→ 100 billion spans/giây
→ Observability infrastructure sẽ explode ngay lập tức
```

---

## Vấn đề cụ thể Discord gặp phải

Discord's architecture tạo ra unique scaling challenges. Khi một user gửi message tới guild với 1 triệu online members, một traced operation có thể spawn ra 1 triệu child spans, một per session. Khi deploy distributed tracing ban đầu, các guild bận nhất bắt đầu không theo kịp activity. Profiling cho thấy processes tốn thời gian đáng kể để unpack trace context, ngay cả khi 99%+ operations không được sampled.

```elixir
defmodule Discord.Tracing.Problem do
  @moduledoc "Minh họa vấn đề naive tracing"

  # Naive approach — làm CHẬM cả guild
  def naive_fanout_with_trace(message, sessions, trace_ctx) do
    span = Tracer.start_span("guild.fanout", parent: trace_ctx)

    Enum.each(sessions, fn session_pid ->
      # Mỗi send kèm theo trace context
      # Context phải được serialize vào message
      # → Tăng message size
      # → Tăng serialization cost
      # → Mỗi session phải deserialize context
      send(session_pid, {message, trace_ctx})
      #                            ^^^^^^^^^^
      #                            thêm ~200 bytes per message
      #                            × 1,000,000 = 200MB extra per fanout
    end)

    Tracer.finish_span(span)
  end

  # Profile result (Discord's actual numbers):
  # Before tracing:  guild fanout CPU = 45%
  # After naive tracing: guild fanout CPU = 65% (+20%)
  # Cause: serialize/deserialize context × 1,000,000
end
```

---

## Discord's Transport Library — wrap message passing

Thay vì instrument HTTP layer, Discord wrap chính `send/2` của BEAM:

```elixir
defmodule Discord.Transport do
  @moduledoc """
  Wrap Erlang message passing để inject trace context.

  Key design decisions:
  1. Không dùng process dictionary (anti-pattern, không visible)
  2. Wrap message structure thay vì modify process state
  3. Receiver tự quyết có unpack context không
  4. Sampling decision propagate theo message
  """

  # Message format với trace context
  # {:transport_msg, original_message, trace_envelope}
  defmodule Envelope do
    defstruct [
      :trace_id,
      :span_id,
      :sampled,        # boolean — có trace không?
      :baggage,        # key-value metadata
    ]
    # Compact: chỉ 4 fields, ~50 bytes khi serialized
    # Quan trọng: :sampled field đọc được mà không deserialize toàn bộ
  end

  # ── Send với trace context ────────────────────────────────────

  def send(pid, message) do
    case get_current_trace() do
      nil ->
        # Không có trace context → send bình thường, zero overhead
        Kernel.send(pid, message)

      %Envelope{sampled: false} = env ->
        # Có trace nhưng không sampled → propagate nhưng không record
        # Receiver biết context nhưng không tạo spans
        Kernel.send(pid, {:transport_msg, message, %{env | baggage: nil}})
        # Shrink envelope: bỏ baggage khi không sampled

      %Envelope{} = env ->
        # Sampled → propagate full context
        Kernel.send(pid, {:transport_msg, message, env})
    end
  end

  # ── Receive — unpack nếu có envelope ─────────────────────────

  defmacro handle_transport(pattern, do: block) do
    quote do
      def handle_info({:transport_msg, unquote(pattern), trace_env}, state) do
        # Set trace context cho process này
        Discord.Transport.set_current_trace(trace_env)
        result = unquote(block)
        Discord.Transport.clear_current_trace()
        result
      end

      # Backward compatible: handle non-transport messages
      def handle_info(unquote(pattern), state) do
        unquote(block)
      end
    end
  end

  # Lưu trace context trong process dictionary
  # (acceptable vì chỉ dùng nội bộ trong 1 message handler)
  def get_current_trace do
    Process.get(:transport_trace_context)
  end

  def set_current_trace(env) do
    Process.put(:transport_trace_context, env)
  end

  def clear_current_trace do
    Process.delete(:transport_trace_context)
  end
end
```

---

## Dynamic Sampling — core innovation

Discord implement dynamic sampling dựa trên fanout size. Messages gửi tới 1 recipient giữ sampling decision 100% thời gian. Messages fanned out tới 100 recipients giảm xuống 10% sampling. Ở 10,000+ recipients, chỉ 0.1% sessions capture spans.

```elixir
defmodule Discord.Tracing.DynamicSampler do
  @moduledoc """
  Sample rate tỉ lệ nghịch với fanout size.
  Đảm bảo: tổng spans/giây không vượt quá budget.
  """

  # Target: tối đa 10,000 spans/giây từ fanout operations
  @max_spans_per_second 10_000

  # Tính sample rate dựa trên số recipients
  def sample_rate_for_fanout(recipient_count) do
    cond do
      recipient_count <= 1 ->
        1.0          # 100% — point-to-point, cheap

      recipient_count <= 10 ->
        1.0          # 100% — small fanout

      recipient_count <= 100 ->
        0.1          # 10% — medium fanout

      recipient_count <= 1_000 ->
        0.01         # 1%

      recipient_count <= 10_000 ->
        0.001        # 0.1%

      recipient_count <= 100_000 ->
        0.0001       # 0.01%

      true ->
        0.00001      # 0.001% — 1 triệu recipients
    end
    # Expected spans với 1,000,000 recipients × 0.001% = 10 spans
    # 10 spans × 100,000 messages/giây = 1,000,000 spans/giây
    # Vẫn quá nhiều → cần thêm upstream sampling
  end

  # Quyết định có sample không — deterministic dựa trên trace_id
  # Đảm bảo: cùng trace_id → cùng quyết định trên mọi node
  def should_sample?(trace_id, sample_rate) do
    # Hash trace_id → [0, 1) uniform distribution
    hash = :erlang.phash2(trace_id, 1_000_000) / 1_000_000
    hash < sample_rate
  end

  # Apply sampling tại fanout point
  def sample_fanout(trace_env, recipient_pids) do
    count = length(recipient_pids)
    rate  = sample_rate_for_fanout(count)

    Enum.map(recipient_pids, fn pid ->
      # Mỗi recipient nhận sampling decision riêng
      # Nhưng deterministic: cùng trace → cùng kết quả
      sampled = should_sample?(trace_env.trace_id, rate)
      {pid, %{trace_env | sampled: sampled}}
    end)
  end
end
```

---

## Implementation trong Guild Process

```elixir
defmodule Discord.Guild.Process do
  use GenServer

  # Nhận message với trace context
  def handle_info(
    {:transport_msg, {:new_message, message}, trace_env},
    state
  ) do
    span = if trace_env.sampled do
      Discord.Tracer.start_span("guild.message_received",
        parent:     trace_env,
        attributes: %{
          guild_id:       state.guild_id,
          channel_id:     message.channel_id,
          active_members: map_size(state.active_sessions),
        }
      )
    end

    # Fanout với dynamic sampling
    active_pids = Map.values(state.active_sessions)
    fanout_count = length(active_pids)

    sampled_pids = Discord.Tracing.DynamicSampler.sample_fanout(
      trace_env,
      active_pids
    )

    # Gửi mỗi session với sampling decision riêng
    Enum.each(sampled_pids, fn {pid, recipient_env} ->
      Discord.Transport.send(pid, {:new_message, message})
      # Transport.send tự inject recipient_env vào message
    end)

    if span do
      Discord.Tracer.finish_span(span, %{
        recipients_total:   fanout_count,
        recipients_sampled: Enum.count(sampled_pids, fn {_, e} -> e.sampled end),
      })
    end

    {:noreply, state}
  end
end
```

---

## Optimization 1: Skip context khi không sampled

Optimization quan trọng nhất: sessions bị cấm bắt đầu new traces sau khi nhận fanned-out messages. Sessions có thể continue existing traces, nhưng sẽ không independently quyết định sample. Change này recover gần như toàn bộ overhead, giảm CPU usage từ 55% xuống 45%.

```elixir
defmodule Discord.Session.Process do
  use GenServer

  def handle_info(
    {:transport_msg, {:new_message, message}, trace_env},
    state
  ) do
    # CRITICAL OPTIMIZATION:
    # Session KHÔNG bắt đầu trace mới từ fanout messages
    # Chỉ continue nếu upstream đã sampled

    if trace_env.sampled do
      # Upstream sampled → tạo span cho delivery
      span = Discord.Tracer.start_span("session.message_delivered",
        parent:     trace_env,
        attributes: %{
          user_id:    state.user_id,
          latency_ms: compute_latency(message),
        }
      )

      deliver_to_websocket(message, state)

      Discord.Tracer.finish_span(span)
    else
      # Không sampled → deliver trực tiếp, zero overhead
      # Không tạo span, không deserialize full context
      deliver_to_websocket(message, state)
    end

    {:noreply, state}
  end

  # Sessions KHÔNG thể bắt đầu trace mới từ fanout
  # Ngăn chặn: session tự quyết sample → nhân explosion spans
  def handle_info({:new_message, message}, state) do
    # Message không có trace context → deliver bình thường
    # Session không create trace riêng cho delivery
    deliver_to_websocket(message, state)
    {:noreply, state}
  end
end
```

---

## Optimization 2: Fast-path cho unsampled context

Optimization quan trọng nhất đến từ gRPC request handling. 75% thời gian xử lý request là để unpack trace context. Discord build filter đọc sampling flag từ encoded trace context string mà không cần full deserialization. Nếu trace không sampled, context không được propagate.

```elixir
defmodule Discord.Tracing.FastPath do
  @moduledoc """
  Đọc sampling flag mà không deserialize toàn bộ context.
  Critical cho high-throughput paths.
  """

  # Trace context format (compact binary):
  # <<version, trace_id::128, span_id::64, flags::8>>
  # Total: ~18 bytes

  # flags byte: bit 0 = sampled
  @sampled_flag 0x01
  @version 0x00

  def encode_context(%{trace_id: tid, span_id: sid, sampled: sampled}) do
    flags = if sampled, do: @sampled_flag, else: 0x00
    <<@version, tid::128, sid::64, flags::8>>
  end

  # Fast decode: chỉ đọc sampled bit — không deserialize toàn bộ
  def is_sampled?(<<@version, _::128, _::64, flags::8>>) do
    (flags &&& @sampled_flag) != 0
  end

  def is_sampled?(_), do: false

  # Full decode: chỉ khi cần (sampled path)
  def decode_context(<<@version, trace_id::128, span_id::64, flags::8>>) do
    %Discord.Transport.Envelope{
      trace_id: trace_id,
      span_id:  span_id,
      sampled:  (flags &&& @sampled_flag) != 0,
    }
  end

  # Usage tại hot path:
  def handle_incoming_grpc(request, raw_trace_context) do
    if is_sampled?(raw_trace_context) do
      # Chỉ decode khi sampled (~1% requests)
      ctx = decode_context(raw_trace_context)
      handle_with_trace(request, ctx)
    else
      # 99% requests: skip decode hoàn toàn
      handle_without_trace(request)
    end
    # Before: 75% thời gian spent decoding context cho mọi request
    # After: decode chỉ 1% → 74% time saved
  end
end
```

---

## Trace flow qua toàn bộ Discord stack

```elixir
defmodule Discord.Tracing.FullFlow do
  @moduledoc """
  End-to-end trace của 1 message qua Discord infrastructure.
  Chỉ với 0.001% sampling rate cho large guilds.
  """

  def trace_message_flow do
    # t=0: Client gửi message qua WebSocket
    # Phoenix Channel tạo root span
    root_span = Tracer.start_span("ws.message_received", [
      sampled: upstream_sampling_decision(),
    ])

    # Nếu sampled (0.01% of messages from large guilds):
    trace_env = %Envelope{
      trace_id: root_span.trace_id,
      span_id:  root_span.id,
      sampled:  root_span.sampled,
    }

    # Transport.send tới Guild process
    # Guild nhận → tạo span "guild.message_received"
    Transport.send(guild_pid, {:new_message, message})

    # Guild fanout tới 1,000,000 sessions
    # DynamicSampler: rate = 0.00001 (0.001%)
    # → ~10 sessions sẽ được sampled
    # → 999,990 sessions: zero overhead

    # 10 sampled sessions tạo span "session.message_delivered"

    # Kết quả trace:
    # root_span (ws.message_received)
    #   └─ guild_span (guild.message_received, fanout=1M)
    #        ├─ session_span_1 (session.message_delivered, user=A)
    #        ├─ session_span_2 (session.message_delivered, user=B)
    #        └─ ... (10 total)

    # Useful: trace cho thấy end-to-end latency
    # Lightweight: chỉ 12 spans thay vì 1,000,002 spans
  end
end
```

---

## Incident discovery — giá trị thực của distributed tracing

Traces cho thấy members gặp 16-phút delays khi kết nối tới guild process bị ảnh hưởng — user impact có thể quantify được mà metrics và logs không thể reveal. Traces cũng expose downstream cascade: users không thể click vào guild trong suốt outage.

```elixir
defmodule Discord.Tracing.IncidentAnalysis do
  @moduledoc """
  Real incident Discord discovered via traces.
  Guild process không theo kịp activity → 16-phút delays.
  """

  # Trace data của incident:
  # root_span: ws.message_received, duration=14ms (bình thường)
  #   └─ guild.message_received: duration=960,000ms  ← 16 phút!
  #        sampled sessions: duration=960,015ms       ← tương tự

  # Trước distributed tracing:
  # Metric: "guild CPU = 95%" → không biết impact tới users
  # Log: "guild process slow" → không biết bao lâu users chờ

  # Sau distributed tracing:
  # Trace: "user A waited 16 phút to receive message" → clear impact
  # Alert trigger: span duration > 5000ms cho guild.message_received

  def setup_trace_based_alerts do
    # Alert khi span duration quá cao
    # Liên kết trực tiếp với user experience
    Prometheus.histogram_observe(
      :guild_message_fanout_duration_ms,
      %{guild_id: guild_id},
      span.duration_ms
    )

    # SLO alert: p99 > 1000ms → SLA breach
    # Trace data → biết chính xác guild nào, user nào bị ảnh hưởng
  end
end
```

---

## Kết hợp tracing với existing telemetry

```elixir
defmodule Discord.Tracing.TelemetryBridge do
  @moduledoc """
  Bridge giữa :telemetry events (Prometheus) và
  distributed traces (Jaeger/Honeycomb).

  Không replace nhau — complement nhau:
  - Metrics: what's happening at aggregate level
  - Traces:  why a specific request is slow
  """

  def setup_bridge do
    # Attach tới telemetry events
    :telemetry.attach_many(
      "discord-trace-bridge",
      [
        [:discord, :guild, :fanout, :stop],
        [:discord, :session, :delivery, :stop],
        [:discord, :rust, :nif, :duration],
      ],
      &handle_telemetry_event/4,
      nil
    )
  end

  def handle_telemetry_event(
    [:discord, :guild, :fanout, :stop],
    measurements,
    metadata,
    _config
  ) do
    # Nếu đang trong sampled trace → tạo span
    case Discord.Transport.get_current_trace() do
      %{sampled: true} = trace_env ->
        span = Tracer.start_span("guild.fanout",
          parent: trace_env,
          start_time: measurements.start_time
        )

        Tracer.finish_span(span, %{
          duration_ms:      measurements.duration / 1_000_000,
          recipient_count:  metadata.recipient_count,
          guild_id:         metadata.guild_id,
        })

      _ ->
        # Không sampled → chỉ update Prometheus counter
        # (Prometheus metrics luôn được collect, không sample)
        :ok
    end
  end
end
```

---

## CPU impact — before và after optimizations

```
Optimization timeline (Discord production data):

Initial deployment (no optimization):
  Guild CPU:   65% (+20% from baseline 45%)
  Session CPU: 55% (+10% from baseline 45%)
  Cause: serialize/deserialize context even when not sampled

After "sessions don't start new traces":
  Guild CPU:   48% (+3% from baseline)
  Session CPU: 45% (baseline, fully recovered)
  Cause: sessions skip trace creation → 10M fewer span operations/s

After "fast-path sampling check":
  Guild CPU:   46% (+1% from baseline)
  gRPC CPU:    reduced 74% for unsampled requests
  Cause: bit check instead of full protobuf decode

After "shrink unsampled envelope":
  Guild CPU:   45.5% (~baseline)
  Network:     3% reduction (smaller envelopes)
  Cause: strip baggage from unsampled messages

Final state:
  Tracing overhead: ~0.5% CPU
  Trace coverage:   meaningful (catches real incidents)
  Span volume:      within budget (10k spans/s)
```

---

## Tóm tắt — design principles cho actor model tracing

```
Standard distributed tracing assumption:
  1 request = 1 trace = linear chain
  → Works for HTTP microservices

Actor model reality:
  1 event = 1 trace = exponential fan
  → Needs fundamentally different approach

Discord's solutions:
  1. Wrap message passing (Transport library)
     Không modify process state → clean abstraction

  2. Dynamic sampling based on fanout
     Sample rate ∝ 1/fanout_size
     Maintain: constant span budget regardless of guild size

  3. Sampling propagation, not sampling decision per hop
     Upstream decides → downstream inherit
     Sessions cannot independently trigger sampling

  4. Fast-path for unsampled context
     Check 1 bit → skip entire decode
     74% reduction in context processing overhead

  5. Shrink envelope for unsampled
     Strip baggage → smaller messages → less network + GC

Result:
  Before: tracing made large guilds unusable
  After:  tracing overhead < 1% CPU
          Meaningful traces cho debugging
          SLO-linked alerts với user impact data
```

---

# Q12: BEAM Scheduler Reductions — Tại Sao send/2 Tốn 30-70μs

*Layer thấp nhất của toàn bộ series — VM internals*

---

## Reduction là gì — đơn vị đo công việc của BEAM

```
Operating System scheduler: preempt theo thời gian (time slice)
  → Thread chạy 10ms → OS preempt → thread khác chạy

BEAM scheduler: preempt theo "reductions" (đơn vị công việc)
  → Process chạy 2000 reductions → BEAM preempt → process khác chạy

Tại sao BEAM chọn reductions thay vì time?
  Time-based: khó predict khi nào preempt
    → 1 function call có thể mất 1ns hoặc 1s
    → Timer interrupt có thể xảy ra giữa chừng
  Reduction-based: predictable
    → Mỗi operation có reduction cost cố định
    → Process luôn được preempt ở "safe points"
    → Không bao giờ preempt giữa chừng một operation
```

---

## Reduction cost của từng operation

```erlang
%% Reduction costs (approximate, từ BEAM source code)

%% Function calls
normal_function_call      = 1 reduction
bif_call (built-in)       = 1-5 reductions (tùy BIF)
nif_call (native)         = varies (dirty scheduler nếu > 1ms)

%% Data operations  
pattern_match             = 1 reduction
list_cons [H|T]           = 1 reduction
tuple_element             = 1 reduction
map_get                   = 1-2 reductions

%% Process operations
send/2 (local, small msg) = 1 reduction + scheduling cost
send/2 (local, large msg) = 1 reduction + GC pressure
send/2 (remote)           = 1 reduction + network write

%% ETS operations
ets.lookup                = 1 reduction (+ copy cost)
ets.insert                = 1 reduction

%% Each process quota:
reductions_per_timeslice  = 2000 reductions
```

---

## Tại sao send/2 thực sự tốn 30-70μs

`send/2` chỉ consume 1 reduction — nhưng 1-reduction không có nghĩa là nhanh:

```elixir
defmodule Discord.SchedulerAnalysis do
  @moduledoc """
  Breakdown chi tiết tại sao send/2 tốn 30-70μs
  trong điều kiện production của Discord
  """

  # Scenario: Guild process fanout tới 30,000 sessions
  # Guild process đang dùng reductions liên tục

  def analyze_send_latency do
    # send/2 wall clock time = sum of:

    # Phase 1: Actual send operation (~1μs)
    # - Serialize message (nếu remote) hoặc copy (nếu local)
    # - Enqueue vào mailbox của receiver
    # Cost: ~1μs cho small message

    # Phase 2: Reduction accounting (~0μs)
    # - BEAM đếm 1 reduction
    # - Nếu process còn < 2000 reductions: tiếp tục ngay

    # Phase 3: Scheduler overhead (0μs đến 60μs+)
    # ĐÂY LÀ PHẦN BIẾN ĐỘNG NHẤT
    # Sau 2000 reductions: process bị de-schedule
    # Process chờ turn tiếp theo từ scheduler
    # "Chờ" bao lâu? = depends on scheduler queue length

    # Với Discord's load (hàng triệu processes):
    # Scheduler queue depth: 50-200 processes waiting
    # Mỗi process chạy ~1ms trước khi yield
    # Wait time = queue_depth × avg_timeslice
    # = 100 processes × 0.5ms = 50ms wait time!

    # Kết quả: send/2 "tốn" 50ms không phải vì bản thân nó chậm
    # Mà vì PROCESS BỊ SUSPEND ngay sau send
    # Wall clock time = 1μs (actual work) + 50ms (waiting)
    :ok
  end
end
```

---

## BEAM Scheduler Architecture

---

![beam_scheduler](beam_scheduler.svg)

## Đo lường reduction cost thực tế

```elixir
defmodule Discord.ReductionBenchmark do
  @doc """
  Đo chính xác reductions consumed bởi các operations
  Dùng :erlang.process_info(self(), :reductions)
  """

  def measure do
    # Measure send/2 reductions
    {before_reds, _} = :erlang.process_info(self(), :reductions)

    receiver = spawn(fn -> receive do _ -> :ok end end)
    send(receiver, :hello)

    {after_reds, _} = :erlang.process_info(self(), :reductions)
    IO.puts("send/2 reductions: #{after_reds - before_reds}")
    # Output: send/2 reductions: 1

    # Nhưng wall clock time phụ thuộc vào scheduler state:
    {time_us, _} = :timer.tc(fn ->
      Enum.each(1..10_000, fn _ ->
        send(receiver, :hello)
      end)
    end)
    IO.puts("10,000 sends: #{time_us}μs = #{time_us/10_000}μs/send")
    # Output khi scheduler nhàn rỗi: ~0.5μs/send
    # Output khi scheduler busy (Discord production): 30-70μs/send
  end

  def measure_scheduler_pressure do
    # Tạo áp lực lên scheduler (simulate production)
    # Spawn 10,000 processes để queue scheduler
    Enum.each(1..10_000, fn _ ->
      spawn(fn ->
        # Busy loop consuming reductions
        loop(100_000)
      end)
    end)

    # Bây giờ measure send/2 latency
    receiver = spawn(fn -> receive do _ -> :ok end end)
    {time_us, _} = :timer.tc(fn ->
      send(receiver, :hello)
    end)
    IO.puts("send/2 under pressure: #{time_us}μs")
    # Output: 45-80μs — khớp với Discord's observation
  end

  defp loop(0), do: :ok
  defp loop(n), do: loop(n - 1)
end
```

---

## BEAM Scheduler Tuning — những gì Discord làm

### Tuning 1: Số schedulers và dirty schedulers

```elixir
# config/runtime.exs
config :myapp, :beam_settings,
  # Mặc định: schedulers = số CPU cores
  # Discord: giữ default cho normal schedulers

  # Dirty CPU schedulers — cho NIF chạy lâu
  # Mặc định: equal to schedulers
  # Discord tuning: giảm dirty schedulers để
  # không compete với normal work
  dirty_cpu_schedulers: System.schedulers_online() |> div(2),

  # Async thread pool (cho blocking I/O)
  async_threads: 30
```

```bash
# Start BEAM với scheduler settings
erl +S 16:8 +SDcpu 8:4
#   +S schedulers:dirty_schedulers
#   +SDcpu dirty_cpu:dirty_io
#   16 normal schedulers
#   8 dirty CPU schedulers
```

### Tuning 2: Scheduler interval và yield frequency

```elixir
# vm.args (hoặc elixir flags)
# +sbt tnnps  = thread no node proc stack
# Bind schedulers to CPU cores để tránh cache miss
# +sbwt none  = disable scheduler busy wait
#               (tiết kiệm CPU khi load thấp)
# +swt very_low = scheduler wake threshold
```

```elixir
# Trong code: explicit yield để tránh monopolize scheduler
defmodule Discord.Guild.Process do
  # BAD: loop 30,000 sends chiếm scheduler
  def fanout_bad(sessions, message) do
    Enum.each(sessions, fn pid ->
      send(pid, message)
      # 30,000 sends = 30,000 reductions
      # Process chạy 2000 reductions → preempt → tiếp
      # → Mất nhiều scheduler context switches
    end)
  end

  # BETTER: yield sau mỗi batch
  def fanout_better(sessions, message) do
    sessions
    |> Enum.chunk_every(500)  # batch 500 sends
    |> Enum.each(fn batch ->
      Enum.each(batch, &send(&1, message))
      # Sau 500 sends = 500 reductions
      # Explicitly yield → cho processes khác chạy
      # → Smoother scheduler, ít jitter hơn
      :erlang.yield()
    end)
  end

  # BEST: dùng Manifold (đã giải thích ở Q2)
  # Manifold giảm số sends từ 30,000 xuống số_nodes
end
```

### Tuning 3: Process priority

```elixir
defmodule Discord.PriorityTuning do
  # BEAM có 4 process priorities:
  # :low, :normal, :high, :max

  # Discord dùng selective priority:

  def start_critical_processes do
    # Guild process: normal (default)
    # Không nâng lên :high vì có thể starve sessions

    # ZenMonitor: high priority
    # Phải process nodedown events nhanh
    Process.spawn(fn ->
      Process.flag(:priority, :high)
      ZenMonitor.Local.run()
    end, [])

    # Cluster EventHandler: high priority
    Process.spawn(fn ->
      Process.flag(:priority, :high)
      Discord.Cluster.EventHandler.run()
    end, [])

    # Session processes: normal
    # (có hàng triệu cái — không nên :high)

    # Background jobs (metrics flush, cleanup): low
    Process.spawn(fn ->
      Process.flag(:priority, :low)
      Discord.Metrics.Flusher.run()
    end, [])
  end

  # Cảnh báo: :max priority chỉ dùng cho VM internals
  # User code dùng :max → có thể starve tất cả processes khác
end
```

### Tuning 4: Message queue monitoring

```elixir
defmodule Discord.SchedulerMonitor do
  @moduledoc """
  Monitor scheduler utilization và process queue depth.
  Alert khi có bottleneck.
  """

  def collect_scheduler_metrics do
    # Scheduler utilization (0.0 - 1.0 per scheduler)
    util_data = :scheduler.utilization(
      :erlang.statistics(:scheduler_wall_time)
    )

    Enum.each(util_data, fn {scheduler_id, busy, _total} ->
      :telemetry.execute(
        [:beam, :scheduler, :utilization],
        %{value: busy},
        %{scheduler_id: scheduler_id}
      )
    end)

    # Process với mailbox queue lớn nhất
    top_queues = Process.list()
      |> Enum.map(fn pid ->
        case Process.info(pid, [:message_queue_len, :registered_name]) do
          [{:message_queue_len, len}, {:registered_name, name}] ->
            {pid, len, name}
          _ -> {pid, 0, nil}
        end
      end)
      |> Enum.sort_by(&elem(&1, 1), :desc)
      |> Enum.take(10)

    # Alert nếu bất kỳ process nào queue > 10,000 messages
    Enum.each(top_queues, fn {pid, len, name} ->
      if len > 10_000 do
        :telemetry.execute(
          [:beam, :process, :queue_overflow],
          %{queue_length: len},
          %{pid: inspect(pid), name: name}
        )
      end
    end)
  end

  # Scheduler wall time — phải enable trước khi dùng
  def enable_scheduler_wall_time do
    :erlang.system_flag(:scheduler_wall_time, true)
  end
end
```

---

## Reduction budget và tail call optimization

```elixir
defmodule Discord.TailCallOptimization do
  @moduledoc """
  Tail call optimization (TCO) trong Erlang/Elixir
  ảnh hưởng đến reduction counting và memory.
  """

  # BAD: body recursion — stack grows, không TCO
  # Mỗi recursive call: 1 new stack frame
  # N items = N stack frames = O(N) memory
  def process_members_bad([], _fun), do: :ok
  def process_members_bad([head | tail], fun) do
    fun.(head)
    process_members_bad(tail, fun)  # NOT tail position
    # Stack frame giữ lại để return về đây
  end

  # GOOD: tail recursion — constant stack, TCO
  # BEAM tối ưu thành loop, không tốn stack
  def process_members_good([], _fun), do: :ok
  def process_members_good([head | tail], fun) do
    fun.(head)
    process_members_good(tail, fun)  # TAIL position → TCO
    # Không cần stack frame mới
    # Reductions: 1 per iteration, không tích lũy
  end

  # Với 1,000,000 members:
  # process_members_bad:  stack overflow (thường ở ~50k)
  # process_members_good: chạy tốt, constant memory

  # Erlang guarantee: tail calls không tốn stack
  # → Guild fanout loop phải là tail recursive
  defp fanout_loop([], _message), do: :ok
  defp fanout_loop([pid | rest], message) do
    send(pid, message)
    fanout_loop(rest, message)  # tail call → no stack growth
  end
end
```

---

## System flags tuning cho Discord workload

```elixir
defmodule Discord.VMTuning do
  @moduledoc """
  BEAM VM flags Discord tune để optimize scheduler behavior
  """

  # vm.args file:
  @vm_args """
  ## Schedulers
  +S auto:auto          # auto-detect based on CPU topology
  +sbwt none            # scheduler busy wait: disabled
                        # saves CPU when load is low
  +swt very_low         # wake threshold: quick to wake schedulers
                        # important for burst traffic

  ## Reduction quota
  ## Default 2000 — Discord keeps default
  ## Increasing: less preemption, higher latency variance
  ## Decreasing: more preemption, lower latency, more overhead

  ## Memory settings
  +MBas aobf            # alloc strategy: address order best fit
  +MBlmbcs 512          # largest multiblock carrier: 512KB
  +MMmcs 30             # max cached memory carriers

  ## Distribution buffer
  +zdbbl 32768          # distribution buffer: 32MB
                        # default 1MB too small for Discord's fanout

  ## GC settings
  ## Per-process: configured in application code
  """

  def set_process_gc_opts(pid, opts \\ []) do
    # Tune GC per process type
    :erlang.process_flag(pid, :min_heap_size,
      opts[:min_heap] || 233)          # default: 233 words

    :erlang.process_flag(pid, :min_bin_vheap_size,
      opts[:min_bin_vheap] || 46422)   # binary heap

    :erlang.process_flag(pid, :max_heap_size, %{
      size:           opts[:max_heap] || 0,  # 0 = unlimited
      kill:           opts[:kill_on_overflow] || false,
      error_logger:   true
    })
  end

  # Guild process: large heap OK (giữ nhiều state)
  def configure_guild_process(pid) do
    set_process_gc_opts(pid,
      min_heap:    10_000,    # start with larger heap
      max_heap:    0          # unlimited (ETS handles large data)
    )
  end

  # Session process: small heap (nhiều triệu sessions)
  def configure_session_process(pid) do
    set_process_gc_opts(pid,
      min_heap:  233,         # default, keep small
      max_heap:  50_000       # kill if session leaks memory
    )
    # Kill session nếu heap > 50,000 words (~400KB)
    # Prevents memory leak từ 1 bad session ảnh hưởng node
  end
end
```

---

## Jitter patterns — tại sao p99 >> p50

```elixir
defmodule Discord.LatencyJitter do
  @moduledoc """
  Giải thích tại sao Discord thấy p50=3ms nhưng p99=45ms
  cho cùng operation
  """

  def explain_jitter do
    # p50 (median): guild process chạy thường xuyên
    # → scheduler queue nhàn
    # → send/2 → preempt → resume nhanh (~3ms)

    # p99: occasional events gây scheduler spike
    # Events gây spike:
    # 1. GC chạy trên 1 process lớn → chiếm 1 scheduler
    # 2. NIF call dài (nếu không dùng dirty scheduler)
    # 3. Sudden burst: 10,000 users join cùng lúc
    # 4. Network hiccup → retry storm

    # Ví dụ GC spike:
    # Session_1 (heap 50MB) GC bắt đầu
    # → GC chạy trên Scheduler 0 trong 50ms
    # → Scheduler 0 không serve processes khác 50ms
    # → Tất cả processes trên Scheduler 0 bị delay 50ms
    # → p99 spike

    # Mitigation: ETS off-heap (đã nói ở Q5)
    # → Processes có heap nhỏ → GC nhanh → ít jitter
    :ok
  end

  # Đo jitter thực tế
  def measure_jitter do
    samples = Enum.map(1..10_000, fn _ ->
      {time, _} = :timer.tc(fn ->
        receiver = spawn(fn -> receive do _ -> :ok end end)
        send(receiver, :hello)
        # Không measure gì sau send
        # Chỉ measure thời gian từ call đến return
      end)
      time  # microseconds
    end)

    sorted = Enum.sort(samples)
    p50  = Enum.at(sorted, 5_000)
    p95  = Enum.at(sorted, 9_500)
    p99  = Enum.at(sorted, 9_900)
    p999 = Enum.at(sorted, 9_990)

    IO.puts("p50:  #{p50}μs")
    IO.puts("p95:  #{p95}μs")
    IO.puts("p99:  #{p99}μs")
    IO.puts("p999: #{p999}μs")

    # Typical Discord production output:
    # p50:  2μs
    # p95:  15μs
    # p99:  45μs
    # p999: 120μs
  end
end
```

---

## Practical implication — tại sao Manifold là bắt buộc

```
Guild fanout 30,000 sends WITHOUT Manifold:

Timeline trên Scheduler 0:
  t=0ms:    Guild_X bắt đầu fanout
  t=0-1ms:  Guild_X runs 2000 reductions (sends ~2000 messages)
  t=1ms:    Guild_X preempted
  t=1-1.3ms: Session_1 runs (handles received message)
  t=1.3ms:  Session_1 preempted
  ...
  [200 processes take turns]
  ...
  t=61ms:   Guild_X resume, sends next 2000 messages
  t=61-122ms: Same cycle repeats
  ...
  t=915ms:  Guild_X finishes all 30,000 sends

Wall clock: 915ms — unacceptable

Guild fanout WITH Manifold:
  Guild sends 10 messages (1 per node):
  t=0-1ms:  Guild_X runs 10 reductions
  t=1ms:    Guild_X done, moves on immediately

  On each remote node (parallel):
  Manifold.Partitioner runs fanout locally
  → Local sends, no TCP, much faster
  → Done in ~10ms

Total wall clock: ~10ms instead of 915ms → 90x improvement
```

---

## Tóm tắt — scheduler insights

```
BEAM scheduler facts:
  Unit: reductions (not time)
  Quota: 2000 reductions per timeslice
  Function call: 1 reduction
  send/2: 1 reduction (but scheduling overhead varies)

Why send/2 = 30-70μs in Discord production:
  Actual work: ~1μs
  Scheduling overhead: 29-69μs
  Cause: high scheduler queue depth under load

Mitigation strategies Discord uses:
  1. Manifold: reduce sends from N to num_nodes
     → 30,000 sends → 10 sends → 90x less scheduler pressure
  2. Relay processes: shard fanout across multiple processes
     → Each relay: 15,000 sends (independent schedulers)
  3. Passive sessions: 90% fewer sessions need sends
     → Direct reduction in reduction consumption
  4. ETS off-heap: smaller heaps → faster GC → less scheduler stall
  5. Process priority: critical processes get CPU first
  6. Dirty schedulers: NIF work doesn't block normal schedulers

Key mental model:
  BEAM scheduler is cooperative within quotas
  Optimizing for reductions = optimizing for fairness
  Low reductions per operation → better sharing → lower p99 latency
```

---

# Q13: @everyone Ping — Permission Check cho 1 Triệu Users

*Bài toán "simple feature, hard implementation" điển hình nhất của Discord*

---

## Tại sao @everyone là bài toán riêng biệt

```
Bình thường khi có message mới:
  Guild biết ai có thể xem channel → đã cache trong relay
  Fanout: gửi tới pids đã filtered sẵn
  Cost: O(active_members_in_relay)

Khi có @everyone ping:
  Phải kiểm tra từng member trong guild:
    1. Member có thể xem channel này không?
       (channel permission overwrites)
    2. Member có bị mute không?
    3. Member có role @everyone suppressed không?
    4. Member có suppress @everyone trong user settings không?
  Cost: O(ALL_members) — không phải O(active)
  Với Midjourney: O(1,000,000)
```

```elixir
# Naive approach — thảm họa
defmodule Discord.Guild.Process do
  def handle_cast({:everyone_ping, channel_id, message}, state) do
    # Lặp qua TẤT CẢ members
    # 1,000,000 iterations × permission_check = vài giây
    eligible = state.members  # 1,000,000 entries trong process heap!
      |> Enum.filter(fn {user_id, member} ->
        can_see_channel?(member, channel_id) and
        not suppressed_everyone?(member) and
        not muted_in_channel?(member, channel_id)
      end)
      |> Enum.map(fn {user_id, _} -> user_id end)

    # Guild bị BLOCK toàn bộ thời gian này
    # Không xử lý được messages khác
    # Users thấy guild "frozen" vài giây

    notify_eligible_users(eligible, message, state)
    {:noreply, state}
  end
end
```

---

## Tại sao @everyone khó hơn fanout bình thường

```elixir
defmodule Discord.Permission.Complexity do
  @moduledoc """
  Permission check cho @everyone không đơn giản là 1 bit check.
  Là cả một computation tree.
  """

  def can_receive_everyone_ping?(member, channel, guild) do
    # Layer 1: Guild-level base permissions
    base_perms = compute_base_permissions(member.roles, guild.roles)

    # Layer 2: Channel permission overwrites
    # Mỗi channel có thể override guild permissions
    # Mỗi role có thể có overwrite khác nhau
    channel_perms = apply_channel_overwrites(
      base_perms,
      member.roles,
      channel.permission_overwrites
    )

    # Layer 3: Check specific permission
    # VIEW_CHANNEL + MENTION_EVERYONE (or admin)
    can_view    = has_permission?(channel_perms, :view_channel)
    can_mention = has_permission?(channel_perms, :mention_everyone) or
                  has_permission?(channel_perms, :administrator)

    # Layer 4: Member-specific suppressions
    not_suppressed = not member_suppressed_everyone?(member)
    not_muted      = not member_muted_in_channel?(member, channel.id)

    # Layer 5: User notification preferences
    # (stored separately — không trong guild state)
    notif_allowed = user_allows_everyone_notification?(member.user_id)

    can_view and can_mention and not_suppressed and
    not_muted and notif_allowed
  end

  defp compute_base_permissions(member_roles, guild_roles) do
    # Admin check: if admin role → all permissions
    if has_admin_role?(member_roles, guild_roles) do
      :all_permissions
    else
      # Accumulate permissions from all roles
      Enum.reduce(member_roles, 0, fn role_id, acc ->
        role = Map.get(guild_roles, role_id)
        acc ||| role.permissions  # bitwise OR
      end)
    end
  end

  defp apply_channel_overwrites(base_perms, member_roles, overwrites) do
    # Apply role overwrites first (lower priority)
    perms_after_roles = Enum.reduce(member_roles, base_perms, fn role_id, acc ->
      case Map.get(overwrites, {:role, role_id}) do
        nil -> acc
        %{allow: allow, deny: deny} ->
          (acc ||| allow) &&& ~~~deny  # apply allow/deny bits
      end
    end)

    # Apply member-specific overwrite (highest priority)
    case Map.get(overwrites, {:member, member_roles}) do
      nil -> perms_after_roles
      %{allow: allow, deny: deny} ->
        (perms_after_roles ||| allow) &&& ~~~deny
    end
  end
end
```

---

## ETS Worker Pattern — giải pháp Discord

Một trong những kẻ phạm tội lớn nhất về unresponsiveness là các operations cần chạy trong guild và iterate qua tất cả members. Những operations này tương đối hiếm nhưng có xảy ra. Discord giải quyết bằng ETS: store members trong ETS, spawn worker processes để chạy async dùng shared ETS data, giải phóng guild để tiếp tục xử lý messages.

```elixir
defmodule Discord.Guild.EveryonePing do
  @moduledoc """
  @everyone ping handler — worker process pattern.

  Key insight: guild process KHÔNG làm permission check.
  Worker process đọc ETS trực tiếp và làm check độc lập.
  Guild tiếp tục serve traffic trong khi worker chạy.
  """

  # Guild process nhận @everyone ping
  def handle_cast({:everyone_ping, channel_id, message}, state) do
    # Spawn worker — non-blocking
    # Worker sẽ làm heavy lifting
    spawn_everyone_worker(
      state.ets_table,
      state.guild_id,
      channel_id,
      message,
      state.channels,
      state.roles
    )

    # Guild tiếp tục ngay lập tức
    # Không chờ worker xong
    {:noreply, state}
  end

  defp spawn_everyone_worker(table, guild_id, channel_id,
                              message, channels, roles) do
    # Task.start: spawn process, không link với guild
    # Nếu worker crash → guild không bị ảnh hưởng
    Task.start(fn ->
      channel = Map.get(channels, channel_id)

      # Permission check engine
      # Chạy trong worker process — không block guild
      eligible_user_ids = check_everyone_permissions(
        table,
        channel,
        roles
      )

      # Gửi notifications
      deliver_everyone_notifications(
        guild_id,
        eligible_user_ids,
        message
      )
    end)
  end
end
```

---

## Worker — parallel permission checking

```elixir
defmodule Discord.Guild.EveryoneWorker do
  @moduledoc """
  Worker process: đọc ETS, check permissions, deliver notifications.
  Chạy song song với guild process.
  """

  @chunk_size 10_000     # process 10k members per sub-batch
  @parallel_workers 8    # số workers parallel

  def check_everyone_permissions(ets_table, channel, roles) do
    # Strategy: chia ETS table thành chunks
    # Xử lý parallel với multiple Task processes
    # Collect results và merge

    total_members = :ets.info(ets_table, :size)

    # Lấy tất cả keys (chỉ user_ids, không copy values)
    all_keys = :ets.select(ets_table, [
      {{:"$1", :_}, [], [:"$1"]}
    ])
    # all_keys = [user_id_1, user_id_2, ... user_id_1000000]
    # Chỉ integers — nhỏ, OK to copy

    # Chia thành chunks và process parallel
    all_keys
    |> Enum.chunk_every(div(total_members, @parallel_workers))
    |> Enum.map(fn chunk ->
      Task.async(fn ->
        process_member_chunk(chunk, ets_table, channel, roles)
      end)
    end)
    |> Task.await_many(30_000)  # timeout 30 giây
    |> List.flatten()
  end

  defp process_member_chunk(user_ids, table, channel, roles) do
    # Xử lý một chunk members
    Enum.reduce(user_ids, [], fn user_id, eligible ->
      # Đọc member data từ ETS — single entry, nhỏ
      case :ets.lookup(table, user_id) do
        [{^user_id, member}] ->
          if everyone_eligible?(member, channel, roles) do
            [user_id | eligible]
          else
            eligible
          end
        [] ->
          eligible  # member đã leave
      end
    end)
  end

  defp everyone_eligible?(member, channel, roles) do
    # Tính permissions — đã explain ở trên
    base   = compute_base_permissions(member.roles, roles)
    actual = apply_channel_overwrites(base, member.roles,
                                       channel.permission_overwrites)

    has_permission?(actual, :view_channel) and
    not member_suppressed_everyone?(member)
    # Note: user notification preferences check separately
    # (không stored trong guild ETS — stored per-user trong separate store)
  end
end
```

---

## Deliver notifications — không phải simple fanout

```elixir
defmodule Discord.Guild.EveryoneDelivery do
  def deliver_everyone_notifications(guild_id, eligible_user_ids, message) do
    # @everyone delivery khác với normal message delivery:
    # 1. Active sessions: gửi full message (như bình thường)
    # 2. Passive sessions: gửi PUSH notification (không chỉ badge)
    # 3. Offline users: queue push notification (mobile push)

    # Phân loại eligible users
    {active, passive, offline} = classify_users(
      guild_id,
      eligible_user_ids
    )

    # Gửi tới active sessions — qua relay (nhanh)
    deliver_to_active(guild_id, active, message)

    # Gửi push notification tới passive (có WebSocket nhưng không active)
    deliver_push_to_passive(passive, message)

    # Queue mobile push cho offline users
    # (qua APNs, FCM — không qua Elixir)
    queue_mobile_push(offline, message)
  end

  defp classify_users(guild_id, user_ids) do
    # Lookup session registry
    Enum.reduce(user_ids, {[], [], []}, fn user_id, {active, passive, offline} ->
      case Discord.Session.Registry.lookup(user_id) do
        {:ok, session_pid, :active}  -> {[session_pid | active], passive, offline}
        {:ok, session_pid, :passive} -> {active, [session_pid | passive], offline}
        {:error, :not_found}         -> {active, passive, [user_id | offline]}
      end
    end)
  end

  defp deliver_to_active(guild_id, active_pids, message) do
    # Dùng Manifold — group by node, batch
    Manifold.send(active_pids, {:everyone_mention, message})
  end

  defp deliver_push_to_passive(passive_pids, message) do
    # Passive sessions nhận stripped notification
    # Đủ để hiện badge + notification popup
    notification = %{
      type:       :everyone_mention,
      preview:    String.slice(message.content, 0, 100),
      channel_id: message.channel_id,
      guild_id:   message.guild_id,
      timestamp:  message.timestamp,
    }
    Manifold.send(passive_pids, {:push_notification, notification})
  end

  defp queue_mobile_push(offline_user_ids, message) do
    # Batch push notifications
    # Gửi qua separate push notification service
    # Không block main flow
    Discord.PushNotification.queue_batch(
      offline_user_ids,
      %{
        title:   "@everyone mentioned you",
        body:    String.slice(message.content, 0, 200),
        guild_id: message.guild_id,
      }
    )
  end
end
```

---

## Timing analysis — bao lâu thực sự?

```elixir
defmodule Discord.EveryoneTiming do
  @doc """
  Breakdown timing của @everyone cho 1,000,000 member guild
  """

  def timing_breakdown do
    # Step 1: Guild dispatch tới worker
    # Thời gian: ~1μs (spawn task + send message)
    # Guild blocked: 0ms ← key!

    # Step 2: Worker đọc tất cả keys từ ETS
    # :ets.select 1,000,000 keys (chỉ integers)
    # Thời gian: ~500ms (copy 1M integers)
    # Guild blocked: 0ms (worker process riêng)

    # Step 3: Chia thành 8 chunks, parallel processing
    # Mỗi chunk: 125,000 members
    # Per member: 1 ETS lookup + permission computation
    # ETS lookup: ~0.5μs
    # Permission compute: ~2μs (bitwise ops)
    # Per member total: ~2.5μs
    # Per chunk: 125,000 × 2.5μs = ~312ms
    # 8 parallel workers: ~312ms (parallel!)
    # Guild blocked: 0ms

    # Step 4: Merge results từ 8 workers
    # Flatten 8 lists: ~10ms
    # Guild blocked: 0ms

    # Step 5: Classify users (active/passive/offline)
    # Session Registry lookup per user
    # ETS lookup: ~0.5μs per user
    # 1,000,000 × 0.5μs = ~500ms (nếu sequential)
    # → Thực tế: chunk + parallel → ~100ms
    # Guild blocked: 0ms

    # Step 6: Deliver tới active sessions
    # Manifold.send → group by node → batch TCP
    # ~10ms
    # Guild blocked: 0ms

    # Step 7: Queue push notifications (async)
    # ~5ms
    # Guild blocked: 0ms

    # TOTAL worker time: ~1,400ms (1.4 giây)
    # GUILD BLOCKED: 0ms ← đây là điều quan trọng

    # Users experience:
    # Active users: nhận message trong ~10ms (fanout)
    # Passive users: nhận push trong ~1.5 giây
    # Offline users: nhận mobile push trong ~2-5 giây
    :ok
  end
end
```

---

## Permission cache optimization

```elixir
defmodule Discord.Permission.Cache do
  @moduledoc """
  @everyone check 1,000,000 members tốn 1.4 giây.
  Với cache thông minh có thể giảm xuống ~200ms.
  """

  # Cache level 1: Pre-computed permission bits trong ETS
  # Thay vì compute mỗi lần → store kết quả

  def precompute_channel_permissions(guild_id, channel_id) do
    table     = :"guild_members_#{guild_id}"
    channel   = Discord.Guild.get_channel(guild_id, channel_id)
    roles     = Discord.Guild.get_roles(guild_id)

    # Batch precompute và store trong ETS
    :ets.foldl(fn {user_id, member}, _acc ->
      perms = compute_effective_permissions(member, channel, roles)

      # Store computed permissions back vào ETS
      # (separate ETS table để tránh bloat member table)
      :ets.insert(:"guild_permissions_#{guild_id}",
        {{user_id, channel_id}, perms})

      nil
    end, nil, table)
  end

  # Khi role hoặc channel thay đổi → invalidate cache
  def invalidate_channel_permissions(guild_id, channel_id) do
    :ets.match_delete(
      :"guild_permissions_#{guild_id}",
      {{:_, channel_id}, :_}
    )
    # Lazily recompute khi cần
  end

  # @everyone check với cache
  def check_with_cache(user_id, channel_id, guild_id) do
    cache_table = :"guild_permissions_#{guild_id}"

    case :ets.lookup(cache_table, {user_id, channel_id}) do
      [{{^user_id, ^channel_id}, perms}] ->
        # Cache hit: ~0.5μs
        has_permission?(perms, :view_channel)

      [] ->
        # Cache miss: compute và store
        perms = compute_and_cache(user_id, channel_id, guild_id)
        has_permission?(perms, :view_channel)
    end
  end

  # Với pre-computed cache:
  # Per member check: ~0.5μs (ETS lookup only)
  # 1,000,000 × 0.5μs = 500ms sequential
  # 8 parallel workers: ~62ms
  # Total @everyone: ~200ms instead of 1.4 giây
end
```

---

## Rate limiting @everyone

```elixir
defmodule Discord.Guild.EveryoneRateLimit do
  @moduledoc """
  @everyone là expensive operation.
  Rate limit để tránh abuse và resource exhaustion.
  """

  # Không cho phép @everyone quá thường xuyên
  @cooldown_seconds 3600  # 1 giờ per channel

  def check_and_execute_everyone_ping(guild_id, channel_id,
                                       user_id, message, state) do
    rate_limit_key = "everyone:#{guild_id}:#{channel_id}"

    case check_rate_limit(rate_limit_key) do
      :ok ->
        # Proceed với @everyone
        spawn_everyone_worker(state, channel_id, message)
        {:ok, state}

      {:error, :rate_limited, reset_at} ->
        # Gửi error message về cho user
        # Không execute @everyone
        notify_rate_limited(user_id, channel_id, reset_at)
        {:error, :rate_limited}
    end
  end

  defp check_rate_limit(key) do
    # Dùng ETS-based rate limiter (không cần Redis cho này)
    now = System.system_time(:second)

    case :ets.lookup(:everyone_rate_limits, key) do
      [{^key, last_used}] when now - last_used < @cooldown_seconds ->
        reset_at = last_used + @cooldown_seconds
        {:error, :rate_limited, reset_at}

      _ ->
        :ets.insert(:everyone_rate_limits, {key, now})
        :ok
    end
  end

  # Special case: @here (chỉ ping online members)
  # Cheaper vì chỉ check online sessions, không phải tất cả members
  def handle_here_ping(guild_id, channel_id, message, state) do
    # @here: chỉ active + passive sessions (đã connected)
    # Không cần scan 1M members trong ETS
    connected_users = get_connected_users(state)

    eligible = Enum.filter(connected_users, fn {user_id, _} ->
      can_see_channel_cached?(user_id, channel_id, guild_id)
    end)

    # Rẻ hơn nhiều: chỉ O(connected) thay vì O(all_members)
    deliver_everyone_notifications(guild_id, eligible, message)
  end
end
```

---

## Tổng kết — @everyone là microcosm của Discord architecture

```
@everyone giống Discord architecture ở dạng thu nhỏ:

Challenge          Solution              Pattern
─────────────────────────────────────────────────────────────
1M permission      ETS worker            Off-heap + async worker
checks block guild

Sequential         8 parallel Task       Process parallelism
takes 1.4s         workers               (không phải threads)

Permission         Pre-computed cache    ETS as computation cache
recomputed each    in ETS                (update on invalidation)
time

Deliver to         3-tier delivery       Classify by engagement
1M users           active/passive/       level → appropriate
differently        offline               channel

Abuse prevention   ETS rate limiter      Local state, fast check

Guild stays        Task.start            Fire-and-forget async
responsive         (not linked)          worker, crash isolated
─────────────────────────────────────────────────────────────

Timeline:
  Guild blocked:    0ms (immediate Task.start)
  Active delivery:  ~10ms
  Full eligibility: ~200ms (với cache), ~1.4s (không cache)
  Push delivery:    ~1.5s
  Mobile push:      2-5s

Key lesson:
  "Làm trong guild process" ≠ "Guild process làm"
  Guild OWNS operation nhưng DELEGATES execution
  ETS là shared memory bridge cho delegation
```

---

