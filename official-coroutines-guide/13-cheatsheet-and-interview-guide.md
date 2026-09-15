# Bài 13 — Sổ Tay Tra Cứu & 20 Câu Hỏi Phỏng Vấn Hóc Búa (Master Cheatsheet & Interview Guide)

> **Tài liệu tham chiếu:** [Kotlin Coroutines Guide — JetBrains Official](https://kotlinlang.org/docs/coroutines-guide.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Cung cấp bảng tra cứu nhanh toàn diện (Master Cheatsheet) cho mọi thành phần trong Kotlin Coroutines & Flow, kèm theo bộ 20 câu hỏi phỏng vấn chuyên sâu (Senior / Lead level) có lời giải chi tiết, phân tích sâu cơ chế máy ảo JVM (State Machine, CPS, Memory Footprint).

---

## PHẦN 1: MASTER CHEATSHEET (SỔ TAY TRA CỨU NHANH)

### 1.1 Coroutine Builders & Scopes
| Builder / Scope | Kiểu trả về | Cơ chế xử lý ngoại lệ | Kịch bản sử dụng |
| :--- | :--- | :--- | :--- |
| **`launch`** | `Job` | Tự động lan truyền lên cha ngay lập tức (Crash nếu không có handler). | Tác vụ "Fire-and-forget" (ghi log, cập nhật DB, gửi analytic). |
| **`async`** | `Deferred<T>` | Đóng gói ngoại lệ vào `Deferred`, chỉ ném ra khi gọi `await()`. | Tính toán song song để lấy kết quả (Parallel decomposition). |
| **`runBlocking`** | `T` | Chặn đứng thread hiện tại cho đến khi coroutine hoàn thành. | Hàm `main()` kiểm thử, cầu nối với mã nguồn đồng bộ Java cũ. |
| **`coroutineScope`** | `T` | Hai chiều: 1 con lỗi $\rightarrow$ hủy toàn bộ anh em và hủy chính scope. | Gom nhóm các tác vụ phụ thuộc lẫn nhau. |
| **`supervisorScope`**| `T` | Một chiều: 1 con lỗi $\rightarrow$ cô lập lỗi, không làm sập scope và anh em. | Giao diện UI Android, Server xử lý request độc lập. |

---

### 1.2 Dispatchers Matrix
| Dispatcher | Thread Pool bên dưới | Giới hạn số Threads | Công việc phù hợp |
| :--- | :--- | :--- | :--- |
| **`Dispatchers.Main`** | UI Event Loop (Android Looper, Swing EDT) | 1 thread | Thao tác UI, cập nhật State, lắng nghe sự kiện bấm. |
| **`Dispatchers.Main.immediate`** | UI Event Loop | 1 thread | Bỏ qua việc đẩy vào hàng đợi nếu coroutine đang ở sẵn Main thread. |
| **`Dispatchers.Default`** | Shared Worker Pool | Số lõi CPU (tối thiểu 2) | Tính toán nặng: Parse JSON lớn, mã hóa, xử lý mảng, thuật toán. |
| **`Dispatchers.IO`** | Elastic On-demand Pool | Mặc định 64 (hoặc số lõi CPU nếu lớn hơn) | I/O chặn: Đọc/ghi File, gọi API Network, truy vấn SQLite/Room. |
| **`Dispatchers.Unconfined`** | Thread hiện tại của người gọi | Không cố định (nhảy theo điểm resume) | Viết unit test nhanh, hoặc tác vụ siêu nhẹ không chạm shared state. |

---

### 1.3 Flow Operators Cheatsheet
- **Biến đổi**: `.map { }`, `.transform { emit(...) }`, `.mapNotNull { }`
- **Lọc**: `.filter { }`, `.take(n)` (hủy bằng `CancellationException`), `.drop(n)`, `.distinctUntilChanged()`
- **Đệm & Backpressure**: `.buffer(capacity, onBufferOverflow)`, `.conflate()` (bỏ qua giá trị cũ), `.collectLatest { }` (hủy tác vụ xử lý cũ khi có giá trị mới)
- **Đổi Thread**: `.flowOn(Dispatcher)` (chỉ tác động lên thượng nguồn upstream, hỗ trợ Operator Fusion)
- **Kết hợp**: `.zip()` (ghép cặp 1-1, kết thúc theo luồng ngắn nhất), `.combine()` (lấy giá trị mới nhất của mỗi luồng), `.merge()` (trộn nhiều luồng đồng thời)
- **Làm phẳng**: `flatMapConcat` (tuần tự), `flatMapMerge` (song song đa nguồn), `flatMapLatest` (hủy luồng cũ khi có giá trị mới)
- **Vòng đời**: `.onStart { }`, `.onEach { }`, `.onCompletion { cause -> }`, `.onEmpty { }`
- **Kết thúc (Terminal)**: `.collect()`, `.first()`, `.toList()`, `.reduce()`, `.launchIn(scope)`

---

## PHẦN 2: 20 CÂU HỎI PHỎNG VẤN CHUYÊN SÂU (KÈM LỜI GIẢI CHI TIẾT)

### Câu 1: Hàm `suspend` hoạt động như thế nào bên dưới bytecode JVM?
**Trả lời:**
Dưới bytecode JVM, hàm `suspend` không hề sử dụng bất kỳ cơ chế ảo thuật nào mà được trình biên dịch Kotlin chuyển đổi bằng hai kỹ thuật cốt lõi:
1. **Continuation-Passing Style (CPS)**: Trình biên dịch ngầm bổ sung một tham số cuối cùng vào hàm: `Continuation<T>`, trong đó `Continuation` chứa `CoroutineContext` và hàm `resumeWith(Result<T>)`. Kiểu trả về thực tế của hàm trên JVM trở thành `Any?` để có thể trả về hoặc kết quả tính toán `T`, hoặc hằng số đặc biệt `COROUTINE_SUSPENDED`.
2. **State Machine (Máy trạng thái)**: Toàn bộ thân hàm `suspend` được trình biên dịch biên dịch thành một lớp ẩn kế thừa từ `ContinuationImpl` chứa một biến `label: Int` và một câu lệnh `switch(label)`. Mỗi khi hàm gặp một điểm tạm ngưng (suspension point), nó lưu các biến cục bộ vào trường của lớp, tăng `label` lên 1 và trả về `COROUTINE_SUSPENDED`. Khi tác vụ bất đồng bộ hoàn tất, nó gọi `continuation.resumeWith()` để đưa máy trạng thái nhảy vào đúng `case` tiếp theo mà không cần giữ hay chặn thread.

---

### Câu 2: Sự khác biệt bản chất giữa `coroutineScope` và `supervisorScope` là gì?
**Trả lời:**
- **`coroutineScope`**: Thiết lập ranh giới thất bại **hai chiều (bidirectional failure propagation)**. Nếu bất kỳ một coroutine con nào bên trong ném ra ngoại lệ (không phải `CancellationException`), nó sẽ lập tức hủy coroutine cha và tất cả các coroutine con khác.
- **`supervisorScope`**: Thiết lập ranh giới thất bại **một chiều (unidirectional failure propagation)**. Sử dụng một `SupervisorJob` nội bộ, sự cố thất bại của một nhánh con bị cô lập hoàn toàn, không làm hủy cha và không làm ảnh hưởng đến các nhánh con anh em. Chỉ khi chính bản thân `supervisorScope` bị hủy thì toàn bộ con của nó mới bị hủy theo.

---

### Câu 3: Tại sao `@Volatile` không đủ để đảm bảo an toàn đa luồng cho câu lệnh `counter++`?
**Trả lời:**
`@Volatile` chỉ đảm bảo tính **tuyến tính (linearizable)** cho các thao tác **đọc đơn lẻ** và **ghi đơn lẻ** trên biến bằng cách vô hiệu hóa CPU cache và ép đọc/ghi trực tiếp vào bộ nhớ chính RAM.  
Tuy nhiên, câu lệnh `counter++` là một thao tác phức hợp (compound action) gồm 3 bước riêng biệt:
1. Đọc giá trị hiện tại của `counter` từ RAM.
2. Tăng giá trị lên 1 trong thanh ghi CPU.
3. Ghi giá trị mới ngược lại RAM.  
Nếu hai coroutine chạy trên hai thread khác nhau cùng thực hiện bước 1 tại cùng thời điểm, cả hai sẽ cùng đọc được cùng một giá trị cũ và ghi đè lên nhau, dẫn đến hiện tượng **mất mát cập nhật (lost update)**. Để giải quyết, bắt buộc phải dùng `AtomicInteger` (sử dụng lệnh phần cứng CAS) hoặc non-blocking lock `Mutex.withLock`.

---

### Câu 4: Phân biệt `StateFlow` và `SharedFlow`? Cơ chế Backpressure của chúng ra sao?
**Trả lời:**
- **`StateFlow`**:
  - Luôn có một giá trị hiện tại (`value`), có tính chất **conflation** (tự động gộp dữ liệu, chỉ lưu giá trị mới nhất).
  - So sánh bằng toán tử `equals()`: Nếu gán giá trị mới trùng với giá trị cũ, nó sẽ không phát ra cho subscriber.
  - Luôn tương đương với một `SharedFlow` có cấu hình: `replay = 1`, `onBufferOverflow = BufferOverflow.DROP_OLDEST`.
- **`SharedFlow`**:
  - Không bắt buộc có giá trị ban đầu. Có thể cấu hình số lượng phần tử phát lại (`replay`) cho subscriber mới đến và kích thước bộ đệm thêm (`extraBufferCapacity`).
  - Hỗ trợ đầy đủ các chiến lược tràn đệm: `SUSPEND` (áp dụng backpressure làm ngưng emitter), `DROP_OLDEST`, hoặc `DROP_LATEST`.

---

### Câu 5: Tại sao việc gọi `withContext` bên trong khối `flow { ... }` bị cấm nghiêm ngặt? Giải pháp thay thế là gì?
**Trả lời:**
Flow tuân thủ nghiêm ngặt nguyên lý **Bảo toàn Ngữ cảnh (Context Preservation)**: Việc phát dữ liệu (`emit`) bắt buộc phải diễn ra trong cùng một `CoroutineContext` với coroutine đang gọi `collect()`. Nếu bạn gọi `withContext(Dispatchers.IO)` bên trong `flow { emit(...) }`, bộ máy Flow sẽ phát hiện ra sự thay đổi ngữ cảnh trái phép và ném ngay ngoại lệ:  
`java.lang.IllegalStateException: Flow invariant is violated`.

**Giải pháp Chuẩn:** Sử dụng toán tử **`.flowOn(Dispatchers.IO)`**. Toán tử này chỉ thay đổi ngữ cảnh cho khối thượng nguồn (upstream) đứng trước nó thông qua một channel đệm trung gian mà hoàn toàn không vi phạm ngữ cảnh thu thập ở hạ nguồn.

---

### Câu 6: Phân biệt sự khác nhau giữa `.conflate()` và `.collectLatest { }`?
**Trả lời:**
- **`.conflate()`**: Tác động ở **tầng phát và đệm dữ liệu**. Khi collector xử lý chậm, emitter vẫn phát bình thường nhưng bộ đệm chỉ giữ lại phần tử mới nhất và vứt bỏ các phần tử cũ. **Tuy nhiên, tác vụ mà collector đang xử lý dở dang VẪN ĐƯỢC CHẠY TIẾP TỤC CHO ĐẾN HẾT**. Sau khi xong, collector mới lấy giá trị mới nhất tiếp theo.
- **`collectLatest { }`**: Tác động ở **tầng thu thập**. Ngay thời điểm có một phần tử mới xuất hiện ở thượng nguồn, khối lambda của phần tử trước đó **LẬP TỨC BỊ HỦY BỎ (CANCEL) NGAY TẠI ĐIỂM SUSPEND TIẾP THEO** để bắt đầu xử lý ngay phần tử mới.

---

### Câu 7: Tại sao cài đặt `CoroutineExceptionHandler` vào một coroutine con tạo bởi `launch` lại bị bỏ qua hoàn toàn?
**Trả lời:**
Trong mô hình Đồng thời Có cấu trúc (Structured Concurrency), tất cả các coroutine con khi gặp ngoại lệ chưa bắt đều **ủy quyền (delegate)** việc xử lý lên coroutine cha của chúng, cha ủy quyền tiếp lên ông, cho đến khi chạm tới **Root Coroutine**. Do đó:
- `CoroutineExceptionHandler` cài đặt ở ngữ cảnh coroutine con **hoàn toàn vô tác dụng** và bị bộ máy coroutines bỏ qua.
- `CoroutineExceptionHandler` chỉ phát huy tác dụng khi được cài đặt tại:
  1. Ngữ cảnh của một **Root Coroutine** (như `CoroutineScope.launch(handler)`).
  2. Hoặc một coroutine được khởi chạy trực tiếp bên trong **`supervisorScope`** (vì trong supervisorScope, con không chuyển giao lỗi cho cha).

---

### Câu 8: `async` xử lý ngoại lệ khác gì `launch`? Tại sao bọc `try/catch` quanh `async { }` vẫn có thể làm sập parent coroutine?
**Trả lời:**
- `launch` coi ngoại lệ là **chưa bắt (uncaught)** và ném ra ngay lập tức.
- `async` đóng gói ngoại lệ vào đối tượng `Deferred` và chỉ ném ra khi bạn gọi `deferred.await()`.

**Cạm bẫy sống còn:**  
Khối lệnh:
```kotlin
try {
    val deferred = async { throw Exception() }
} catch (e: Exception) { ... }
```
Sẽ **KHÔNG THỂ** bắt được ngoại lệ, bởi vì `async` chỉ bắt đầu chạy bất đồng bộ trong nền. Ngay khi coroutine con của `async` bị lỗi, nó sẽ **lập tức hủy luôn coroutine cha** theo cơ chế Structured Concurrency hai chiều trước khi bạn kịp gọi `await()`.  
**Cách giải quyết**: Muốn cô lập lỗi của `async`, bạn phải chạy nó bên trong một `supervisorScope`.

---

### Câu 9: Điều gì xảy ra khi một coroutine bị hủy? Tại sao không được bắt `CancellationException` mà không ném lại?
**Trả lời:**
Khi coroutine bị hủy (`job.cancel()`), bộ máy coroutine sẽ ném ra một ngoại lệ nội bộ mang tên `CancellationException` tại các điểm tạm ngưng (như `delay()`, `yield()`, `await()`).  
Nếu bạn dùng khối `try { ... } catch (e: Exception)` hoặc `catch (e: Throwable)` và nuốt chửng ngoại lệ này (không gọi `throw e`), bạn đã vô tình **ngăn chặn quá trình hủy bỏ của coroutine**, khiến coroutine tiếp tục chạy ngầm, gây rò rỉ bộ nhớ (memory leak) và phá vỡ cơ chế quản lý vòng đời của ứng dụng.

---

### Câu 10: `NonCancellable` là gì? Tại sao TUYỆT ĐỐI KHÔNG ĐƯỢC dùng nó với `launch` hoặc `async`?
**Trả lời:**
- `NonCancellable` là một đối tượng `Job` đặc biệt luôn luôn ở trạng thái kích hoạt (`isActive = true`), được thiết kế **DUY NHẤT** cho việc dọn dẹp tài nguyên bên trong khối `finally`:
  ```kotlin
  finally {
      withContext(NonCancellable) {
          closeDatabaseOrNetwork() // Có thể gọi hàm suspend mà không bị hủy tiếp
      }
  }
  ```
- **Lý do cấm dùng với `launch` hoặc `async`**: `launch(NonCancellable)` sẽ tạo ra một coroutine **hoàn toàn miễn nhiễm với mọi cơ chế hủy bỏ của cha nó**. Khi Scope cha bị hủy (ví dụ ViewModel bị clear), coroutine này vẫn sẽ tiếp tục chạy ngầm vĩnh viễn, phá vỡ hoàn toàn nguyên lý Structured Concurrency.

---

### Câu 11: `Dispatchers.Main` và `Dispatchers.Main.immediate` khác nhau như thế nào?
**Trả lời:**
- **`Dispatchers.Main`**: Luôn đẩy (post) tác vụ thực thi vào hàng đợi tin nhắn của Main Thread (thông qua `Handler.post` trên Android hoặc `SwingUtilities.invokeLater`). Nghĩa là ngay cả khi bạn đang đứng sẵn trên Main Thread, coroutine vẫn phải chờ một chu kỳ lặp tiếp theo của Event Loop.
- **`Dispatchers.Main.immediate`**: Kiểm tra thread hiện tại trước. Nếu coroutine **đang đứng sẵn trên Main Thread**, nó sẽ **thực thi ngay lập tức một cách đồng bộ (synchronously)** mà không cần điều phối hay xếp hàng đợi, giúp tối ưu hóa hiệu năng và tránh hiện tượng giật màn hình (UI flickering).

---

### Câu 12: `Dispatchers.IO` quản lý số lượng threads như thế nào? Cách tạo giới hạn thread tùy biến với `limitedParallelism`?
**Trả lời:**
- `Dispatchers.IO` sử dụng chung Worker Thread Pool với `Dispatchers.Default`, nhưng cho phép mở rộng linh hoạt lên tới tối đa **64 threads** (hoặc bằng số lõi CPU nếu hệ thống có trên 64 lõi).
- Trong các phiên bản Coroutines hiện đại, để giới hạn số lượng thread chạy đồng thời cho một tác vụ cụ thể (ví dụ giới hạn truy cập DB chỉ dùng tối đa 4 thread để tránh nghẽn connection pool), bạn sử dụng:
  ```kotlin
  val dbDispatcher = Dispatchers.IO.limitedParallelism(4)
  ```
  Hàm này tạo một bộ điều phối mới sử dụng chung pool của `Dispatchers.IO` nhưng đảm bảo không bao giờ có quá 4 coroutine cùng chạy đồng thời.

---

### Câu 13: So sánh 3 toán tử làm phẳng: `flatMapConcat`, `flatMapMerge`, `flatMapLatest`?
**Trả lời:**
Khi biến đổi mỗi phần tử của luồng thành một `Flow` con (`Flow<Flow<T>>`):
1. **`flatMapConcat`**: Xử lý tuần tự. Chờ cho luồng con trước phát hết toàn bộ dữ liệu và kết thúc hoàn toàn mới bắt đầu thu thập luồng con tiếp theo.
2. **`flatMapMerge`**: Xử lý song song. Thu thập đồng thời nhiều luồng con cùng một lúc (có thể cấu hình tham số `concurrency`). Dữ liệu của các luồng con được hòa trộn và phát ra theo thứ tự thời gian thực.
3. **`flatMapLatest`**: Hủy luồng cũ khi có luồng mới. Ngay khi thượng nguồn phát ra phần tử mới, luồng con hiện tại đang chạy sẽ **bị hủy ngay lập tức** để nhường chỗ thu thập luồng con mới.

---

### Câu 14: Kênh `Channel` khác gì với `Flow`? Khi nào nên chọn Channel thay vì Flow?
**Trả lời:**
- **`Flow`**: Là dòng dữ liệu **Nguội (Cold)** (trừ `SharedFlow`/`StateFlow`), mang tính chất **đơn quyền sở hữu (Unicast / 1-to-1)**: Mã nguồn bên trong `flow { ... }` không chạy cho đến khi có người gọi `collect()`. Mỗi collector mới sẽ kích hoạt một pipeline thực thi độc lập.
- **`Channel`**: Là cấu trúc truyền tin **Nóng (Hot)**, mang tính chất **chia sẻ tải (Multicast / Fan-out / Fan-in)**: Dữ liệu được gửi vào Channel bất kể có người nhận hay không. Một phần tử chỉ được tiêu thụ bởi **duy nhất MỘT coroutine** (cơ chế phân chia công việc FIFO).
- **Khi nào chọn Channel**: Khi bạn cần xây dựng các mô hình hàng đợi công việc (Work Queue), phân phối tải công việc nặng cho một nhóm worker coroutine (Fan-out), hoặc gom log/sự kiện từ nhiều nguồn độc lập về một điểm xử lý duy nhất (Fan-in).

---

### Câu 15: Trong mô hình Fan-out của Channel, tại sao bắt buộc phải dùng vòng lặp `for` thay vì hàm mở rộng `consumeEach`?
**Trả lời:**
- Vòng lặp `for (msg in channel)` an toàn tuyệt đối khi nhiều coroutine cùng lắng nghe một channel: Nếu một coroutine worker gặp sự cố lỗi hoặc bị hủy, nó chỉ thoát khỏi vòng lặp của chính nó, các coroutine worker khác vẫn tiếp tục sống và tiêu thụ channel bình thường.
- Hàm mở rộng `consumeEach`: Được thiết kế cho **một người tiêu thụ duy nhất**. Ngay khi lambda của `consumeEach` kết thúc (dù kết thúc bình thường hay ném ngoại lệ), nó sẽ **tự động gọi lệnh hủy (cancel) toàn bộ Channel gốc bên dưới**, làm sập ngay lập tức toàn bộ các coroutine worker khác đang cùng lắng nghe channel đó!

---

### Câu 16: Làm thế nào để Unit Test một Coroutine hoặc Flow bị delay 10 giây mà test case chạy trong chưa đầy 10 mili-giây?
**Trả lời:**
Sử dụng hàm **`runTest`** từ thư viện `kotlinx-coroutines-test`.  
`runTest` tự động thay thế bộ điều phối luồng thời gian thực bằng một **`StandardTestDispatcher`** điều khiển bởi **`TestCoroutineScheduler`** (Thời gian ảo - Virtual Time). Khi gặp lệnh `delay(10_000)`, bộ điều phối ảo sẽ không hề làm sleep thread thật mà tự động tua nhanh đồng hồ logic (`advanceTimeBy(10_000)` hoặc tự động tua tới hết), giúp bài test kiểm tra chính xác toàn bộ logic thời gian trễ nhưng thời gian chạy thực tế trên máy chỉ mất vài mili-giây.

---

### Câu 17: Cơ chế "Operator Fusion" trong Flow là gì? Cho ví dụ minh họa?
**Trả lời:**
Operator Fusion (Dung hợp toán tử) là cơ chế tối ưu hóa nội bộ của thư viện `kotlinx.coroutines`. Khi bạn kết nối liên tiếp các toán tử có cùng bản chất cấu hình đệm (chẳng hạn như `.flowOn(...)` kết hợp với `.buffer(...)` hoặc `.conflate()`), thay vì tạo ra nhiều tầng coroutine và nhiều bộ đệm Channel trung gian lồng nhau gây lãng phí bộ nhớ và chi phí chuyển ngữ cảnh, bộ máy Flow sẽ **tự động gộp (fuse) các toán tử này lại để dùng chung MỘT bộ đệm Channel duy nhất**.

---

### Câu 18: `ThreadLocal` được bảo toàn và chuyển giao giữa các thread trong Coroutines như thế nào?
**Trả lời:**
Do coroutine có thể tạm ngưng trên thread này và phục hồi trên một thread khác trong thread pool, các biến Java `ThreadLocal` thông thường sẽ bị mất hoặc đọc nhầm giá trị của coroutine khác.  
Để giải quyết, Kotlin Coroutines cung cấp hàm mở rộng **`asContextElement()`**:
```kotlin
val myThreadLocal = ThreadLocal<String>()
launch(myThreadLocal.asContextElement(value = "user_123")) {
    // Giá trị user_123 luôn được khôi phục chính xác trên thread thực thi
}
```
Cơ chế: Mỗi khi coroutine được phân phối vào một thread để chạy, context element này sẽ lưu giá trị cũ của thread đó và gán giá trị của coroutine vào; khi coroutine tạm ngưng, nó khôi phục lại giá trị ban đầu cho thread.

---

### Câu 19: Tính thiên vị (Bias) của biểu thức `select` là gì? Cách tận dụng nó?
**Trả lời:**
- **Tính thiên vị**: Trong khối `select { ... }`, nếu tại cùng một thời điểm có **nhiều mệnh đề đều đã sẵn sàng** (ví dụ cả hai Channel đều có dữ liệu), thì mệnh đề **được khai báo đầu tiên theo thứ tự từ trên xuống dưới trong mã nguồn sẽ LUÔN ĐƯỢC CHỌN**.
- **Cách tận dụng**: Bạn có thể xây dựng cơ chế **Kênh chính - Kênh phụ (Fallback / Primary-Secondary Routing)**:
  ```kotlin
  select<Unit> {
      primaryChannel.onSend(data) {}   // Luôn được ưu tiên nếu kênh chính rảnh
      secondaryChannel.onSend(data) {} // Chỉ được chọn khi kênh chính đã đầy
  }
  ```

---

### Câu 20: Kỹ thuật Stack Trace Recovery giải quyết bài toán gì và cách áp dụng cho Custom Exception?
**Trả lời:**
- **Bài toán**: Khi một coroutine nhận ngoại lệ từ một coroutine bất đồng bộ khác (qua `await()`), stack trace gốc của JVM chỉ hiển thị nơi lỗi được sinh ra ở coroutine con mà hoàn toàn biến mất dấu vết dòng code đã gọi `await()` ở coroutine cha.
- **Giải pháp**: Bật chế độ Debug (`-Dkotlinx.coroutines.debug`), thư viện sẽ tự động sao chép Exception và đính kèm stack frame của coroutine cha ngăn cách bởi ranh giới `_COROUTINE._BOUNDARY._`.
- **Áp dụng cho Custom Exception**: Kế thừa interface **`StackTraceRecoverable<E>`** trong thư viện chuẩn Kotlin và override hàm:
  ```kotlin
  override fun copyForStackTraceRecovery(): MyCustomException =
      MyCustomException(customField, message, this)
  ```
