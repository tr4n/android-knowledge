# Bài 01 — Coroutines Basics: suspend, CoroutineScope, Dispatcher & Structured Concurrency

> **Module:** 1 — Kotlin Coroutines & Flow Foundation  
> **Prerequisite:** Kotlin cơ bản (OOP, Higher-order functions, Lambdas)  
> **Official Docs:**
> - [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
> - [Kotlin Coroutines on Android — Android Developers](https://developer.android.com/kotlin/coroutines)
> - [Coroutines Best Practices — Android Developers](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

### 1.1 Kotlin Coroutine là gì?

Theo tài liệu chính thức từ Kotlin & Android Developers:

> *"Coroutines are light-weight threads. Like threads, coroutines can run in parallel, wait for each other, and communicate. The biggest difference is that coroutines are very cheap, almost free: we can create thousands of them, and pay very little in terms of performance."*

Về mặt bản chất kỹ thuật:
- Coroutine **không phải là Thread của Hệ điều hành (OS Thread)**.
- Coroutine là một **luồng thực thi trừu tượng ở cấp độ người dùng (User-space Computations)** có khả năng tạm dừng (**suspend**) tại một thời điểm nhất định và tiếp tục (**resume**) sau đó trên một Thread khác mà **không hề block (khóa) Thread ban đầu**.

---

### 1.2 Các khái niệm và thuật ngữ cốt lõi

```
┌────────────────────────────────────────────────────────────────────────┐
│                          COROUTINE ECOSYSTEM                           │
│                                                                        │
│   CoroutineScope (Quản lý vòng đời & Structured Concurrency)           │
│         │                                                              │
│         ├── CoroutineContext (Tập hợp các element cấu hình)            │
│         │     ├── Job (Vòng đời, trạng thái hủy, quan hệ cha-con)      │
│         │     ├── CoroutineDispatcher (Phân phối Thread thực thi)      │
│         │     ├── CoroutineExceptionHandler (Bắt uncaught exceptions)  │
│         │     └── CoroutineName (Đặt tên phục vụ debugging/logging)    │
│         │                                                              │
│         └── Coroutine Builders (Khởi tạo coroutine)                    │
│               ├── launch { }  ──► Trả về Job (Fire & Forget)           │
│               └── async { }   ──► Trả về Deferred<T> (Trả kết quả)    │
└────────────────────────────────────────────────────────────────────────┘
```

#### `suspend` Keyword
- Một từ khóa modifier đánh dấu hàm có thể bị **tạm dừng (suspended)** mà không làm tắc nghẽn thread hiện tại.
- Hàm `suspend` chỉ có thể được gọi từ một hàm `suspend` khác hoặc từ bên trong một Coroutine Builder (`launch`, `async`).

#### `CoroutineScope`
- Interface định nghĩa phạm vi tồn tại của coroutine:
  ```kotlin
  public interface CoroutineScope {
      public val coroutineContext: CoroutineContext
  }
  ```
- Mọi coroutine bắt buộc phải được khởi tạo trong một `CoroutineScope` để đảm bảo **Structured Concurrency** (tính đồng thời có cấu trúc). Khi scope bị hủy (`cancel`), toàn bộ coroutines con bên trong nó sẽ bị hủy theo.

#### `CoroutineContext`
- Một bảng ánh xạ dạng tập hợp các phần tử (Indexed set of `Element`) cấu hình môi trường thực thi của coroutine. Toán tử `+` được nạp chồng (overloaded) để gộp các context elements:
  ```kotlin
  val context = Dispatchers.IO + Job() + CoroutineName("DownloadWorker")
  ```

#### Bảng thành phần CoroutineContext:

| Thành phần | Kiểu (Type) | Nhiệm vụ chính |
|---|---|---|
| **`Job`** | `Job` | Điều khiển vòng đời, theo dõi trạng thái (`isActive`, `isCompleted`, `isCancelled`), thiết lập quan hệ cha-con. |
| **`Dispatcher`** | `CoroutineDispatcher` | Quyết định coroutine sẽ được đẩy vào Thread hoặc Thread Pool nào để thực thi. |
| **`Name`** | `CoroutineName` | Gán nhãn tên cho coroutine (rất hữu ích khi đọc thread dump hoặc log cat). |
| **`ExceptionHandler`**| `CoroutineExceptionHandler` | Xử lý các exception không được bắt (`uncaught`) ở root coroutine. |

---

### 1.3 Phân loại CoroutineDispatcher chuẩn mực

Android và Kotlin cung cấp sẵn 4 Dispatcher chuẩn:

| Dispatcher | Thread Pool bên dưới | Use Cases phù hợp | Điều cấm kỵ |
|---|---|---|---|
| **`Dispatchers.Main`** | Main/UI Thread của Android (gắn với `Looper.getMainLooper()`) | Cập nhật UI, gọi Composable functions, các thao tác UI tương tác nhanh. | Cấm thực hiện I/O nặng (đọc file, gọi API, parse JSON lớn) gây ANR (Application Not Responding). |
| **`Dispatchers.IO`** | Elastic Thread Pool (mặc định chia sẻ với Default, mở rộng tối đa 64 threads hoặc số CPU cores) | Disk I/O (Room DB, File, SharedPreferences), Network I/O (Retrofit, Ktor, Sockets). | Không dùng cho thuật toán tính toán CPU nặng (sẽ chiếm dụng thread IO không cần thiết). |
| **`Dispatchers.Default`** | Shared Work-Stealing Pool (kích thước pool bằng số lượng CPU cores vật lý, tối thiểu 2) | Tác vụ tốn CPU: Parse JSON phức tạp, xử lý ảnh bitmap, sắp xếp/lọc mảng hàng chục nghìn phần tử, mã hóa/giải mã. | Không chạy I/O bị block trên Default vì sẽ làm nghẽn toàn bộ luồng CPU của app. |
| **`Dispatchers.Unconfined`** | Bắt đầu chạy ngay trên Thread hiện tại cho đến điểm suspension đầu tiên. Sau khi resume, chạy trên thread của hàm vừa resume. | Viết Unit Test đặc thù, hoặc các thao tác không cần dispatch chi phí thấp. | **Không dùng trong Android Production Code** vì luồng chạy không dự đoán trước được, rất dễ vô tình chạm UI từ Background Thread. |

---

### 1.4 Coroutine Builders: `launch` vs `async`

```
┌──────────────────────────────────────────────┬──────────────────────────────────────────────┐
│                  launch { }                  │                  async { }                   │
├──────────────────────────────────────────────┼──────────────────────────────────────────────┤
│ Trả về: Job                                  │ Trả về: Deferred<T> (kế thừa Job)            │
│ Mục đích: Fire-and-forget (bắn và quên)      │ Mục đích: Tính toán song song và trả về data │
│ Nhận kết quả: Không trả về giá trị           │ Nhận kết quả: Gọi .await() để lấy giá trị T  │
│ Xử lý lỗi: Exception bắn thẳng lên Job cha   │ Xử lý lỗi: Exception được đóng gói, chỉ bắn  │
│            ngay khi xảy ra                   │            khi gọi .await()                  │
└──────────────────────────────────────────────┴──────────────────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Continuation-Passing Style (CPS) & State Machine

Khi Kotlin Compiler gặp một hàm có từ khóa `suspend`, nó không giữ nguyên hàm đó mà thực hiện kỹ thuật **CPS Transformation** (Biến đổi sang phong cách truyền Continuation).

#### Mã nguồn Kotlin ban đầu:
```kotlin
suspend fun fetchUserData(userId: String): UserProfile {
    val token = fetchAuthToken()         // Suspension Point 1
    val profile = requestProfile(token)  // Suspension Point 2
    return profile
}
```

#### Mã dịch bytecode (Tương đương Java sau decompilation):
Compiler chèn thêm một tham số ngầm định `Continuation<T>` vào cuối hàm và thay đổi kiểu trả về thành `Any?`:
```java
// Khái niệm interface Continuation
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}

// Hàm được compiler sinh ra dưới dạng State Machine
public Object fetchUserData(String userId, Continuation<? super UserProfile> $completion) {
    // Lớp nội bộ lưu trạng thái giữa các lần suspend
    class FetchUserDataContinuation extends ContinuationImpl {
        Object result;
        int label; // Quản lý trạng thái hiện tại (State Machine)
        Object L$0; // Lưu biến cục bộ token
        
        FetchUserDataContinuation(Continuation completion) {
            super(completion);
        }
        
        @Override
        public Object invokeSuspend(Object result) {
            this.result = result;
            this.label |= Integer.MIN_VALUE;
            return fetchUserData(null, this);
        }
    }
    
    FetchUserDataContinuation $continuation;
    if ($completion instanceof FetchUserDataContinuation) {
        $continuation = (FetchUserDataContinuation) $completion;
    } else {
        $continuation = new FetchUserDataContinuation($completion);
    }
    
    switch ($continuation.label) {
        case 0: // Trạng thái khởi đầu
            $continuation.label = 1;
            Object tokenResult = fetchAuthToken($continuation);
            if (tokenResult == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED; // Trả quyền kiểm soát lại cho Caller/Thread!
            }
            // Nếu chạy đồng bộ xong ngay thì rơi xuống case 1
        case 1: // Resume sau khi fetchAuthToken xong
            String token = (String) $continuation.result;
            $continuation.L$0 = token;
            $continuation.label = 2;
            Object profileResult = requestProfile(token, $continuation);
            if (profileResult == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED;
            }
        case 2: // Resume sau khi requestProfile xong
            UserProfile profile = (UserProfile) $continuation.result;
            return profile;
    }
}
```

#### Phân tích cơ chế:
1. Mỗi hàm `suspend` được biên dịch thành một **State Machine**.
2. Biến `label` đóng vai trò con trỏ bước nhảy.
3. Khi gặp điểm dừng (`COROUTINE_SUSPENDED`), hàm chỉ đơn giản là `return` một hằng số đặc biệt (`COROUTINE_SUSPENDED`). Ngăn xếp cuộc gọi (**Call Stack**) được tháo gỡ (unwind), và **Thread vật lý được giải phóng ngay lập tức** để quay lại xử lý các việc khác (như vẽ UI, đo layout).
4. Dữ liệu cục bộ (`token`, `userId`) không nằm trên Thread Stack mà được đóng gói vào đối tượng `Continuation` lưu trữ trên **Heap Memory**.
5. Khi tác vụ I/O hoàn thành ở tầng OS hoặc socket, callback tầng dưới gọi `continuation.resumeWith(result)`, dispatcher sẽ đưa continuation đó vào hàng đợi để gọi lại hàm với `label` tiếp theo!

---

### 2.2 Memory Model: Thread Stack vs Heap Frame

```
MÔ HÌNH BỘ NHỚ TRUYỀN THỐNG (OS THREAD):
┌────────────────────────────────────────────────────────┐
│ OS Thread (Khởi tạo tốn ~1MB Stack Size)                │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Thread Call Stack (Cố định, không co giãn)         │ │
│ │ [Stack Frame 1: main()]                            │ │
│ │ [Stack Frame 2: computeData()]                     │ │
│ │ [Stack Frame 3: sleep() / blocked I/O] ◄── LÃNG PHÍ│ │
│ └────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
  => 10,000 Threads = ~10 GB RAM -> Crash OutOfMemoryError!

MÔ HÌNH KOTLIN COROUTINES (COOPERATIVE USER-SPACE):
┌────────────────────────────────────────────────────────┐
│ Heap Memory                                            │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Continuation Object 1 (~ vài trăm bytes)           │ │
│ │ Continuation Object 2 (~ vài trăm bytes)           │ │
│ │ Continuation Object 100,000 (~ vài chục MBs)       │ │
│ └────────────────────────────────────────────────────┘ │
│                           ▲                            │
│                           │ Được nạp và xử lý linh hoạt│
│                           ▼                            │
│ ┌────────────────────────────────────────────────────┐ │
│ │ OS Thread Pool (Ví dụ: 4 hoặc 8 CPU Threads cố định│ │
│ │ Thread 1: Chạy Coroutine A ──► Suspend ──► Nhận C  │ │
│ │ Thread 2: Chạy Coroutine B ──► Suspend ──► Nhận D  │ │
│ └────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
  => 100,000 Coroutines = Chỉ tốn vài chục MB RAM -> Hoạt động mượt mà!
```

---

### 2.3 Sơ đồ Phân cấp Job (Job Hierarchy & Cancellation Cascading)

Một trong những nền tảng quan trọng nhất của Structured Concurrency là **cây phả hệ Coroutine (Parent-Child Tree)**:

```
                          Root CoroutineScope
                         (ví dụ: viewModelScope)
                                    │
                       ┌────────────┴────────────┐
                       │   Parent Job (Root)     │
                       └────────────┬────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
      Child Job 1                                       Child Job 2
   (launch: Fetch User)                             (launch: Fetch Orders)
           │                                                 │
     ┌─────┴─────┐                                     ┌─────┴─────┐
     ▼           ▼                                     ▼           ▼
Grandchild A  Grandchild B                        Grandchild C  Grandchild D
```

#### Quy tắc lan truyền (Propagation Rules):
1. **Hủy bỏ (Cancellation) chảy từ TRÊN XUỐNG DƯỚI:**
   Nếu `Parent Job` bị hủy (ví dụ: người dùng thoát màn hình khiến `viewModelScope` gọi `cancel()`), **tất cả Child Jobs và Grandchild Jobs lập tức bị hủy**.
2. **Ngoại lệ (Exception) chảy từ DƯỚI LÊN TRÊN (với Job thường):**
   Nếu `Grandchild A` gặp lỗi không bắt được (`RuntimeException`), nó sẽ hủy chính nó -> ném lên hủy `Child Job 1` -> ném lên hủy `Parent Job` -> dẫn đến `Child Job 2` và các nhánh khác **bị hủy toàn bộ**!
3. **SupervisorJob chặn đứng việc lan truyền lỗi lên trên:**
   Nếu `Parent Job` sử dụng `SupervisorJob()`, khi `Child Job 1` bị crash, lỗi **không** làm hủy `Parent Job`, do đó `Child Job 2` vẫn tiếp tục chạy bình thường!

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Lịch sử xử lý bất đồng bộ trên Android

| Kỷ nguyên | Công nghệ sử dụng | Nhược điểm chí mạng |
|---|---|---|
| **Android 1.0 - 3.0** | `java.lang.Thread`, `Handler`, `Looper` | Quản lý thủ công, tốn bộ nhớ, dễ memory leak khi giữ tham chiếu Activity/Context. |
| **Android 3.0 - 9.0** | `AsyncTask` | Dễ gây Memory Leak do inner class giữ outer Activity reference; hành vi luồng thay đổi qua các phiên bản Android; cancel nửa vời; bị Google chính thức **Deprecated hoàn toàn**. |
| **Android 6.0 - 10.0** | `RxJava 2/3` | Cực kỳ mạnh mẽ nhưng dốc học tập (learning curve) quá cao; hàng trăm toán tử khó nhớ; stack trace khó hiểu khi debug; cần dispose thủ công với `CompositeDisposable`. |
| **Hiện đại (Modern)** | **Kotlin Coroutines** | Tích hợp ở cấp độ ngôn ngữ; cú pháp tuần tự tự nhiên; an toàn vòng đời với Structured Concurrency; stack trace rõ ràng. |

---

### 3.2 Vấn nạn "Callback Hell" vs Cú pháp tuần tự (Sequential Imperative)

#### Cách làm cũ với Callbacks (Pyramid of Doom):
```kotlin
// ANTI-PATTERN: Rất khó đọc, khó handle error, không thể cancel đồng bộ
fetchUser(userId, object : UserCallback {
    override fun onSuccess(user: User) {
        fetchAccountBalance(user.accountId, object : BalanceCallback {
            override fun onSuccess(balance: Balance) {
                fetchTransactionHistory(user.accountId, object : HistoryCallback {
                    override fun onSuccess(history: List<Transaction>) {
                        runOnUiThread {
                            updateUI(user, balance, history)
                        }
                    }
                    override fun onError(e: Exception) { /* Handle error 3 */ }
                })
            }
            override fun onError(e: Exception) { /* Handle error 2 */ }
        })
    }
    override fun onError(e: Exception) { /* Handle error 1 */ }
})
```

#### Giải quyết triệt để với Coroutines:
```kotlin
// CLEAN & MAINTAINABLE: Đọc như code đồng bộ, tự động bảo vệ lifecycle
viewModelScope.launch {
    try {
        val user = repository.fetchUser(userId)
        // Chạy song song 2 tác vụ độc lập để tối ưu thời gian
        val balanceDeferred = async { repository.fetchAccountBalance(user.accountId) }
        val historyDeferred = async { repository.fetchTransactionHistory(user.accountId) }
        
        val balance = balanceDeferred.await()
        val history = historyDeferred.await()
        
        _uiState.value = UiState.Success(user, balance, history)
    } catch (e: Exception) {
        if (e is CancellationException) throw e // Không nuốt CancellationException!
        _uiState.value = UiState.Error(e.message ?: "Đã có lỗi xảy ra")
    }
}
```

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Khi nào NÊN dùng và KHÔNG NÊN dùng?

#### NÊN DÙNG:
1. **Mọi tác vụ I/O:** Gọi Retrofit API, truy vấn Room Database, đọc ghi DataStore / SharedPreferences, truy cập file system.
2. **Tính toán nặng trên nền:** Xử lý chuỗi, mã hóa dữ liệu, giải nén zip, convert Bitmap (`Dispatchers.Default`).
3. **Quản lý trạng thái UI:** Kết hợp với ViewModel và Jetpack Compose.
4. **Tác vụ tuần tự hoặc song song phức tạp:** Cần phối hợp kết quả từ nhiều nguồn dữ liệu.

#### KHÔNG NÊN DÙNG:
1. **Tác vụ nền chạy độc lập khi ứng dụng đã đóng (Background Service lâu dài):** Coroutine gắn với `viewModelScope` hoặc `lifecycleScope` sẽ bị hủy khi app tắt. **Phải dùng WorkManager** (WorkManager hỗ trợ `CoroutineWorker` rất hoàn hảo).
2. **Thao tác đơn giản, tức thì trên bộ nhớ (In-memory field access):** Không cần tạo coroutine cho các phép gán biến hay tính toán toán học đơn giản.

---

### 4.2 Các nguyên tắc vàng (Google Android Architecture Best Practices)

#### 1. Luôn biến suspend function thành "Main-Safe"
Một hàm `suspend` phải luôn an toàn khi được gọi từ Main thread. Nếu hàm làm việc nặng, **chính hàm đó phải chịu trách nhiệm tự chuyển thread** bằng `withContext`:
```kotlin
// ĐÚNG: Caller ở ViewModel không cần quan tâm hàm này chạy thread nào, an toàn tuyệt đối
class UserRepositoryImpl(
    private val api: UserApi,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {
    override suspend fun getUserProfile(id: String): UserProfile = withContext(ioDispatcher) {
        // Chắc chắn đang chạy trên IO Thread Pool
        api.fetchProfile(id).toDomainModel()
    }
}
```

#### 2. Luôn Inject CoroutineDispatcher qua Constructor
Tuyệt đối không hardcode `Dispatchers.IO` bên trong class:
```kotlin
// ANTI-PATTERN: Không thể thay thế khi viết Unit Test
class MyRepository {
    suspend fun loadData() = withContext(Dispatchers.IO) { ... }
}

// BEST PRACTICE: Có thể inject StandardTestDispatcher trong Unit Test
class MyRepository(
    private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun loadData() = withContext(ioDispatcher) { ... }
}
```

#### 3. Tuyệt đối không dùng `GlobalScope`
`GlobalScope` tạo ra các coroutine hoạt động trên phạm vi toàn bộ Application Lifecycle. Chúng không thể bị cancel tự động, dễ dẫn đến rò rỉ bộ nhớ (memory leak) và tiêu hao tài nguyên pin khi màn hình đã đóng. Thay vào đó, luôn dùng:
- `viewModelScope` trong ViewModel.
- `lifecycleScope` trong Activity/Fragment.
- Hoặc một custom `CoroutineScope` gắn với vòng đời component tương ứng.

---

### 4.3 Các Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Nuốt chửng `CancellationException`
Kotlin Coroutines sử dụng ngoại lệ đặc biệt `CancellationException` để báo hiệu việc hủy coroutine. Nếu dùng block `catch (e: Exception)` tổng quát mà không rethrow nó, coroutine sẽ không thể dừng lại!

```kotlin
// SAI (ANTI-PATTERN):
try {
    delay(1000)
} catch (e: Exception) {
    Log.e("Error", "Failed") // Đã nuốt mất CancellationException!
}

// ĐÚNG:
try {
    delay(1000)
} catch (e: CancellationException) {
    throw e // BẮT BUỘC ném lại để coroutine hủy đúng cách
} catch (e: Exception) {
    Log.e("Error", "Business Error: ${e.message}")
}
```

#### Cạm bẫy 2: Gọi `runBlocking` trên Main Thread
`runBlocking` sẽ chặn hoàn toàn thread hiện tại cho đến khi coroutine bên trong hoàn thành. Nếu gọi trên Android Main Thread, giao diện sẽ đơ và hệ thống sẽ hiển thị hộp thoại **ANR (Application Not Responding)** sau 5 giây.
> **Quy tắc:** Chỉ sử dụng `runBlocking` trong Unit Tests hoặc trong hàm `main()` của các ứng dụng Console, **tuyệt đối không dùng trong code UI Android**.

#### Cạm bẫy 3: Vòng lặp tính toán không có điểm nhường (Non-cooperative Loop)
Coroutine chỉ có thể bị hủy nếu nó chủ động hợp tác (cooperative). Nếu có một vòng lặp CPU không chứa suspend function nào, coroutine sẽ không bao giờ dừng dù scope đã bị cancel:

```kotlin
// SAI: Tiếp tục chạy ngầm vô tận dù màn hình đã đóng
viewModelScope.launch(Dispatchers.Default) {
    while (true) {
        // Tính toán ma trận nặng...
    }
}

// ĐÚNG: Kiểm tra isActive hoặc gọi ensureActive() / yield()
viewModelScope.launch(Dispatchers.Default) {
    while (isActive) { // Hoặc gọi ensureActive() ở mỗi vòng lặp
        // Tính toán ma trận...
        yield() // Chủ động nhường quyền cho các coroutine khác cùng thread
    }
}
```

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Chúng ta sẽ xây dựng tính năng **Tải thông tin người dùng và thống kê giao dịch song song**, tuân thủ Clean Architecture từ Data Layer đến ViewModel và UI Layer (Jetpack Compose).

### Bước 1: Khai báo Dependencies (Gradle)

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    // Kotlin Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    // Android Lifecycle & ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.4")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")

    // Jetpack Compose BOM
    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")

    // Unit Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
}
```

---

### Bước 2: Data Layer (Clean Architecture & Main-safe)

```kotlin
// 1. Domain Entities
data class UserProfile(val id: String, val name: String, val email: String)
data class UserStats(val totalOrders: Int, val loyaltyPoints: Int)

// 2. Repository Interface
interface UserRepository {
    suspend fun getUserProfile(userId: String): UserProfile
    suspend fun getUserStats(userId: String): UserStats
}

// 3. Repository Implementation với IO Dispatcher Injection
class UserRepositoryImpl(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {

    override suspend fun getUserProfile(userId: String): UserProfile = withContext(ioDispatcher) {
        // Giả lập network delay 1000ms
        delay(1000)
        UserProfile(id = userId, name = "Alex Johnson", email = "alex@example.com")
    }

    override suspend fun getUserStats(userId: String): UserStats = withContext(ioDispatcher) {
        // Giả lập network delay 800ms
        delay(800)
        UserStats(totalOrders = 42, loyaltyPoints = 1250)
    }
}
```

---

### Bước 3: ViewModel Layer (UDF & Parallel async/await)

```kotlin
sealed interface UserUiState {
    data object Loading : UserUiState
    data class Success(val profile: UserProfile, val stats: UserStats) : UserUiState
    data class Error(val message: String) : UserUiState
}

class UserViewModel(
    private val userRepository: UserRepository,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) : ViewModel() {

    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUserData(userId: String) {
        // Sử dụng viewModelScope: Tự động hủy khi ViewModel onCleared()
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            try {
                // CHẠY SONG SONG: Tận dụng async để tiết kiệm tổng thời gian
                // Thay vì tốn 1000ms + 800ms = 1800ms, chỉ mất max(1000, 800) = 1000ms!
                val profileDeferred = async { userRepository.getUserProfile(userId) }
                val statsDeferred = async { userRepository.getUserStats(userId) }

                val profile = profileDeferred.await()
                val stats = statsDeferred.await()

                _uiState.value = UserUiState.Success(profile, stats)
            } catch (e: CancellationException) {
                // Luôn ném lại CancellationException để Structured Concurrency hoạt động chuẩn
                throw e
            } catch (e: Exception) {
                _uiState.value = UserUiState.Error(e.localizedMessage ?: "Lỗi không xác định")
            }
        }
    }
}
```

---

### Bước 4: UI Layer (Jetpack Compose)

```kotlin
@Composable
fun UserProfileScreen(
    viewModel: UserViewModel,
    modifier: Modifier = Modifier
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        when (val state = uiState) {
            is UserUiState.Loading -> {
                CircularProgressIndicator()
            }
            is UserUiState.Error -> {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Text(
                        text = state.message,
                        color = MaterialTheme.colorScheme.error,
                        style = MaterialTheme.typography.bodyLarge
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { viewModel.loadUserData("USER_123") }) {
                        Text("Thử lại")
                    }
                }
            }
            is UserUiState.Success -> {
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
                ) {
                    Column(modifier = Modifier.padding(16.dp)) {
                        Text(text = state.profile.name, style = MaterialTheme.typography.headlineSmall)
                        Text(text = state.profile.email, style = MaterialTheme.typography.bodyMedium)
                        Divider(modifier = Modifier.padding(vertical = 12.dp))
                        Text(text = "Tổng đơn hàng: ${state.stats.totalOrders}")
                        Text(text = "Điểm tích lũy: ${state.stats.loyaltyPoints}")
                    }
                }
            }
        }
    }
}
```

---

### Bước 5: Viết Unit Test chuẩn mực với `StandardTestDispatcher`

```kotlin
class UserViewModelTest {

    // Test Coroutine Dispatcher cho phép kiểm soát Virtual Time
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var fakeRepository: UserRepository
    private lateinit var viewModel: UserViewModel

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        fakeRepository = object : UserRepository {
            override suspend fun getUserProfile(userId: String) = 
                UserProfile(userId, "Test Name", "test@test.com")
            override suspend fun getUserStats(userId: String) = 
                UserStats(10, 100)
        }
        viewModel = UserViewModel(fakeRepository, testDispatcher)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun loadUserData_success_updatesUiStateToSuccess() = runTest(testDispatcher) {
        // Given
        viewModel.loadUserData("USER_1")
        assertEquals(UserUiState.Loading, viewModel.uiState.value)

        // When: Tua nhanh toàn bộ các suspend function đang chờ
        advanceUntilIdle()

        // Then
        val currentState = viewModel.uiState.value
        assertTrue(currentState is UserUiState.Success)
        val successState = currentState as UserUiState.Success
        assertEquals("Test Name", successState.profile.name)
        assertEquals(10, successState.stats.totalOrders)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Các câu hỏi phỏng vấn Senior/Staff Android

#### Q1: Tại sao `suspend` function không hề làm block Thread? Bản chất OS Thread lúc đó làm gì?
**Trả lời chuẩn bản chất:**
Khi một hàm `suspend` chạm đến điểm dừng (ví dụ chờ socket I/O từ `delay()` hoặc Retrofit), Kotlin compiler kích hoạt cơ chế Continuation Passing Style. Hàm trả về token `COROUTINE_SUSPENDED` và ngay lập tức tháo gỡ stack frame (`return`). Tại tầng OS, thread không bị chuyển sang trạng thái `BLOCKED` hay `WAITING`, mà nó hoàn toàn rảnh rỗi (`RUNNABLE`) để quay lại Event Loop của Dispatcher và thực thi các công việc/coroutine khác. Khi tác vụ I/O xong, OS Interrupt báo về cho Coroutine Runtime gọi `continuation.resumeWith()`, đẩy tiếp phần code còn lại vào hàng đợi của Dispatcher.

#### Q2: Phân biệt bản chất giữa `Job` và `SupervisorJob` trong Structured Concurrency?
**Trả lời chuẩn bản chất:**
- Với **`Job` thông thường**: Có tính đối xứng hai chiều trong lan truyền hủy bỏ. Khi một coroutine con ném ra Exception không được xử lý, nó lập tức thông báo lên Job cha, và Job cha sẽ hủy bỏ tất cả các coroutine anh em khác trong cùng scope.
- Với **`SupervisorJob`**: Tính lan truyền bị chặn theo chiều từ dưới lên (Unidirectional cancellation). Exception của coroutine con không làm ảnh hưởng đến cha hoặc các anh em khác. Đây chính là lý do `viewModelScope` và `lifecycleScope` mặc định sử dụng `SupervisorJob` để một tác vụ con bị lỗi không làm sập toàn bộ ViewModel.

#### Q3: Khi nào nên sử dụng `withContext(NonCancellable)`?
**Trả lời chuẩn bản chất:**
Khi một coroutine đã bị hủy (Cancelled), mọi suspend function được gọi bên trong nó sẽ lập tức ném ra `CancellationException`. Do đó, nếu cần thực hiện các tác vụ dọn dẹp quan trọng bắt buộc phải hoàn thành (như đóng file stream, gửi event log thoát ứng dụng, giải phóng socket) bên trong khối `finally`, ta bắt buộc phải bọc tác vụ đó trong:
```kotlin
withContext(NonCancellable) {
    repository.cleanupResources() // Đảm bảo không bị hủy giữa chừng
}
```

#### Q4: Điều gì xảy ra nếu bạn pass một `Job()` mới vào `viewModelScope.launch(Job())`?
**Trả lời chuẩn bản chất:**
Đây là một **anti-pattern rất nguy hiểm**. Khi truyền một `Job()` mới vào `launch`, bạn đã phá vỡ cây phả hệ (Structured Concurrency) của `viewModelScope`. Coroutine mới này sẽ trở thành một Root Job độc lập, không còn là con của `viewModelScope`. Hậu quả: Khi ViewModel bị hủy (`onCleared()`), coroutine này **sẽ không bị hủy theo**, tiếp tục chạy ngầm gây rò rỉ bộ nhớ và tài nguyên!

---

### 6.2 Lỗi Runtime/Compile thường gặp & Cách khắc phục

| Hiện tượng lỗi | Nguyên nhân gốc rễ | Cách khắc phục triệt để |
|---|---|---|
| **App bị đơ 5s và báo lỗi ANR (Application Not Responding)** | Gọi `runBlocking` hoặc tác vụ I/O/CPU nặng trực tiếp trên `Dispatchers.Main`. | Chuyển tác vụ sang `withContext(Dispatchers.IO)` hoặc `Dispatchers.Default`. Tuyệt đối không dùng `runBlocking`. |
| **Coroutine không dừng lại dù đã gọi `job.cancel()`** | Trong coroutine có vòng lặp tính toán liên tục (`while` / `for`) mà không có bất kỳ suspend function nào để kiểm tra cờ cancellation. | Bổ sung `ensureActive()` hoặc `yield()` bên trong thân vòng lặp. |
| **Mất kết quả khi xoay màn hình điện thoại** | Khởi chạy coroutine trong `lifecycleScope.launch` của Activity nhưng không lưu state vào ViewModel hoặc `rememberSaveable`. | Đưa toàn bộ logic gọi dữ liệu vào `ViewModel` thông qua `viewModelScope` và expose qua `StateFlow`. |
| **Crash: `IllegalStateException: Module with the Main dispatcher is missing` trong Unit Test** | Chạy Unit Test gọi `Dispatchers.Main` mà không cấu hình `kotlinx-coroutines-test`. | Khai báo `Dispatchers.setMain(testDispatcher)` trong `@Before` và `resetMain()` trong `@After`. |

---

*Bài trước: Không có (Bài mở đầu Module 1)*  
*Bài tiếp theo: [02 — Cold Flow vs Hot Flow: StateFlow & SharedFlow](02-cold-flow-vs-hot-flow.md)*
