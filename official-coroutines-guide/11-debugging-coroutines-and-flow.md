# Bài 11 — Gỡ Lỗi & Kiểm Thử Coroutine (Debug Coroutines & Testing)

> **Tài liệu gốc:** [Debug coroutines — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-debugging.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0` & `kotlinx-coroutines-debug:1.11.0`  
> **Mục tiêu:** Nắm vững các công cụ và kỹ thuật gỡ lỗi coroutine trên JVM: kích hoạt Chế độ Debug (`-Dkotlinx.coroutines.debug`); cơ chế khôi phục vết ngăn xếp (Stack Trace Recovery) cho ngoại lệ tùy biến với `StackTraceRecoverable`; theo dõi trạng thái coroutine bằng Debug Agent (`DebugProbes`); phát hiện rò rỉ hoặc nghẽn tác vụ với `CoroutinesTimeout` trong Unit Test; và kiểm thử Flow đường ống với thư viện Turbine.

---

## 1. Thách thức Khi Gỡ lỗi Coroutine (Debugging Challenges)

Việc gỡ lỗi (debugging) các ứng dụng sử dụng coroutine có thể gặp nhiều khó khăn và phức tạp hơn lập trình đa luồng truyền thống vì:
- Nhiều coroutine chạy đồng thời, có thể **tạm ngưng (suspend) trên thread này** nhưng khi tiếp tục lại **phục hồi (resume) trên một thread hoàn toàn khác**.
- Thứ tự thực thi và danh tính các thread được cấp phát có thể thay đổi liên tục giữa các lần chạy, khiến lập trình viên khó lần theo vết dấu vết của một coroutine cụ thể.

Trên nền tảng JVM, thư viện `kotlinx.coroutines` cung cấp 3 tính năng cốt lõi để tháo gỡ khó khăn này:
1. **Chế độ Debug (Debug mode)**: Gán một tên duy nhất cho mỗi coroutine để nhận diện trong debugger và log.
2. **Khôi phục vết ngăn xếp (Stack trace recovery)**: Bổ sung thông tin về nơi coroutine nhận ngoại lệ thay vì chỉ hiển thị nơi ngoại lệ được ném ra ban đầu.
3. **Debug Agent (`kotlinx-coroutines-debug`)**: Theo dõi coroutine khi chúng được tạo, tạm ngưng và phục hồi, đồng thời xuất ra Coroutine Dump toàn diện.

---

## 2. Kích hoạt Chế độ Debug (Enable Debug Mode)

Chế độ Debug gán một định danh tên duy nhất (ví dụ: `@coroutine#1`, `@coroutine#2`) cho mọi coroutine được khởi chạy. Bạn có thể thấy tên coroutine này:
- Trong trình gỡ lỗi Java Debugger của IntelliJ IDEA / Android Studio.
- Trong chuỗi biểu diễn `toString()` của coroutine.
- Trong tên của Thread hệ điều hành trong suốt thời gian thread đó đang thực thi coroutine.

> [!NOTE]
> Chế độ Debug có chi phí runtime cực kỳ nhỏ (negligible runtime overhead). Khi bạn chạy mã với cờ Java Assertions được bật (`-ea`), thư viện `kotlinx.coroutines` sẽ **tự động kích hoạt chế độ Debug**. Các bài kiểm thử Unit test mặc định luôn bật assertions.

### Cách Kích hoạt Thủ công:
Truyền đối số máy ảo JVM sau vào Run/Debug Configuration của IDE hoặc cấu hình Gradle / Maven:
```bash
-Dkotlinx.coroutines.debug
```

#### Các bước thiết lập trong IntelliJ IDEA / Android Studio:
1. Trong thanh công cụ **Run widget**, bấm chọn cấu hình chạy muốn sửa đổi $\rightarrow$ chọn **More Actions** $\rightarrow$ **Edit**.
2. Trong hộp thoại **Run/Debug Configurations**, tại mục **VM options**, nhập: `-Dkotlinx.coroutines.debug`.
3. Bấm **OK** để lưu lại.

---

## 3. Khôi phục Vết Ngăn Xếp (Stack Trace Recovery)

Khi một coroutine nhận một ngoại lệ từ một coroutine khác thông qua hàm suspending (chẳng hạn như `Deferred.await()`), stack trace nguyên bản của ngoại lệ đó **sẽ không chứa các stack frame của coroutine nhận**. Điều này khiến bạn không thể biết hàm `await()` được gọi ở dòng nào trong mã nguồn người gọi, gây trở ngại lớn cho việc chẩn đoán lỗi.

Thư viện `kotlinx.coroutines` giải quyết điều này bằng cơ chế **Khôi phục vết ngăn xếp (Stack trace recovery)**: Nó tạo ra một bản sao của ngoại lệ kèm theo các stack frame bổ sung từ phía coroutine nhận, được ngăn cách rõ ràng bằng ranh giới `_COROUTINE._BOUNDARY._`:

```kotlin
import kotlinx.coroutines.*

object UserProfileService :
    CoroutineScope by CoroutineScope(CoroutineName("UserProfileService")) {

    private fun parseUserProfile(): String {
        error("Invalid user profile")
    }

    private fun loadUserProfile(): String {
        return parseUserProfile()
    }

    // Chạy trong coroutine gọi hàm này
    suspend fun awaitUserProfile() {
        // Khởi chạy một coroutine mới
        val userProfile = async(Dispatchers.Default) {
            // Coroutine mới ném ngoại lệ
            loadUserProfile()
        }

        // Coroutine đang thực thi awaitUserProfile()
        // nhận ngoại lệ thông qua hàm await()
        userProfile.await()
    }
}

suspend fun main() {
    UserProfileService.awaitUserProfile()
}
```

### So sánh Stack Trace Trước và Sau Khi Khôi phục:

**Khi KHÔNG có Stack Trace Recovery (`-Dkotlinx.coroutines.stacktrace.recovery=false`):**
Chỉ thấy điểm ném lỗi trong `parseUserProfile`, hoàn toàn biến mất dấu vết lệnh gọi `userProfile.await()` ở `awaitUserProfile()`:
```none
Exception in thread "main" java.lang.IllegalStateException: Invalid user profile
	at UserProfileService.parseUserProfile(UserProfileService.kt:7)
	at UserProfileService.loadUserProfile(UserProfileService.kt:11)
	at UserProfileService$awaitUserProfile$userProfile$1.invokeSuspend(UserProfileService.kt:17)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
```

**Khi BẬT Chế độ Debug / Stack Trace Recovery:**
Xuất hiện thêm stack frame phía trên ranh giới coroutine, chỉ đích danh dòng gọi `userProfile.await()`:
```none
Exception in thread "main" java.lang.IllegalStateException: Invalid user profile
	at UserProfileService.parseUserProfile(UserProfileService.kt:7)
	at UserProfileService.loadUserProfile(UserProfileService.kt:11)
	at UserProfileService$awaitUserProfile$userProfile$1.invokeSuspend(UserProfileService.kt:17)
	at _COROUTINE._BOUNDARY._(CoroutineDebugging.kt:42)
	at UserProfileService.awaitUserProfile(UserProfileService.kt:21)
	at UserProfileServiceKt.main(UserProfileService.kt:26)
Caused by: java.lang.IllegalStateException: Invalid user profile
	at UserProfileService.parseUserProfile(UserProfileService.kt:7)
	...
```

---

### 3.1 Khôi phục Stack Trace cho Ngoại lệ Tùy biến (`StackTraceRecoverable`)

Stack trace recovery mặc định có thể tự động sao chép các ngoại lệ nếu lớp ngoại lệ đó có constructor công khai nhận tham số `(message, cause)`.

Nếu bạn tự định nghĩa các ngoại lệ tùy biến với các thuộc tính bổ sung (ví dụ số dòng bị lỗi `line: Int`, mã lỗi `errorCode: String`), hãy kế thừa interface **`StackTraceRecoverable`** trong thư viện chuẩn Kotlin:

```kotlin
import kotlinx.coroutines.*
import kotlin.coroutines.ExperimentalStdlibCoroutineSupportApi
import kotlin.coroutines.debug.StackTraceRecoverable

@OptIn(ExperimentalStdlibCoroutineSupportApi::class)
class FileEditException
private constructor(
    val line: Int,
    private val detail: String,
    cause: Throwable?,
) : IllegalStateException("When editing line $line: $detail", cause),
    StackTraceRecoverable<FileEditException> { // Kế thừa interface phục hồi stack trace

    constructor(line: Int, detail: String) : this(line, detail, null)

    // Sao chép lại số dòng, chi tiết và gắn nguyên nhân ban đầu
    override fun copyForStackTraceRecovery(): FileEditException =
        FileEditException(line, detail, this)
}

private fun editFile() {
    throw FileEditException(15, "Unexpected token")
}

suspend fun main() {
    supervisorScope {
        val fileEdit = async(Dispatchers.Default) {
            editFile()
        }
        fileEdit.await()
    }
}
```

---

## 4. Debug Agent: `kotlinx-coroutines-debug`

Module `kotlinx-coroutines-debug` cung cấp một tác nhân JVM chuyên sâu để theo dõi toàn bộ vòng đời coroutine.

### Cài đặt Dependency:
```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-debug:1.11.0")
}
```

### Các Cách Kích Hoạt Debug Agent:
1. Thêm cờ VM Option: `-javaagent:/path/to/kotlinx-coroutines-debug-1.11.0.jar`.
2. Hoặc gọi hàm **`DebugProbes.install()`** ngay khi ứng dụng khởi chạy.

> [!NOTE]
> Bắt đầu từ JDK 21, việc tải động debug agent bằng hàm `DebugProbes.install()` có thể tạo ra cảnh báo từ JVM. Để tránh cảnh báo này, hãy nạp agent thông qua tùy chọn VM `-javaagent`.
> 
> **Lưu ý hiệu năng trong Production:** Nếu bật `DebugProbes` trên môi trường Production, việc thu thập stack trace khi khởi tạo mỗi coroutine mới có thể làm giảm hiệu năng đáng kể. Để khắc phục, hãy thiết lập `DebugProbes.enableCreationStackTraces = false`.

### Các API Chẩn đoán của `DebugProbes`:
- **`DebugProbes.dumpCoroutines()`**: Xuất ra màn hình console danh sách toàn bộ các coroutine đang hoạt động cùng trạng thái và vết ngăn xếp.
- **`DebugProbes.dumpCoroutinesInfo()`**: Trả về danh sách thông tin các coroutine đang hoạt động dưới dạng đối tượng.
- **`DebugProbes.printJob(job)`**: Xuất cây phân cấp coroutine của riêng một `Job` cụ thể.
- **`DebugProbes.printScope(scope)`**: Xuất toàn bộ cây coroutine trực thuộc một `CoroutineScope`.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.debug.*
import kotlin.time.Duration.Companion.seconds

private suspend fun loadAccount() {
    delay(5.seconds)
}

private suspend fun loadPreferences() {
    delay(5.seconds)
}

private suspend fun loadUserProfile() = coroutineScope {
    launch { loadAccount() }
    launch { loadPreferences() }
}

@OptIn(ExperimentalCoroutinesApi::class)
fun main() {
    // Cài đặt debug agent (chỉ cần thiết nếu không dùng VM option -javaagent)
    DebugProbes.install()

    runBlocking {
        val loadingJob = launch {
            loadUserProfile()
        }

        // Đợi một chút để các coroutine con bước vào trạng thái suspend
        delay(1.seconds)

        // In danh sách toàn bộ coroutine đang chạy
        DebugProbes.dumpCoroutines()

        println("============")

        // In cây coroutine của riêng loadingJob
        DebugProbes.printJob(loadingJob)
    }
}
```

**Kết quả Console Output mẫu:**
```none
Coroutines dump 2026/08/18 14:00:08

Coroutine "coroutine#1":BlockingCoroutine{Active}@146ba0ac, state: RUNNING
Coroutine "coroutine#2":StandaloneCoroutine{Active}@4dfa3a9d, state: SUSPENDED
Coroutine "coroutine#3":StandaloneCoroutine{Active}@6eebc39e, state: SUSPENDED
Coroutine "coroutine#4":StandaloneCoroutine{Active}@464bee09, state: SUSPENDED
============
"coroutine#2":StandaloneCoroutine{Active}, continuation is SUSPENDED at line DebugAgentExampleKt$main$1$loadingJob$1.invokeSuspend(DebugAgentExample.kt:27)
	"coroutine#3":StandaloneCoroutine{Active}, continuation is SUSPENDED at line DebugAgentExampleKt$loadUserProfile$2$1.invokeSuspend(DebugAgentExample.kt:14)
	"coroutine#4":StandaloneCoroutine{Active}, continuation is SUSPENDED at line DebugAgentExampleKt$loadUserProfile$2$2.invokeSuspend(DebugAgentExample.kt:15)
```

> [!TIP]
> Module `kotlinx-coroutines-debug` cũng tự động tích hợp sẵn với **BlockHound** để giúp bạn phát hiện ngay lập tức bất kỳ lệnh blocking I/O nào vô tình bị gọi bên trong các ngữ cảnh coroutine non-blocking.

---

## 5. Bắt Lỗi Treo trong Unit Test: `CoroutinesTimeout`

Trong Unit Test, nếu một coroutine bị deadlock hoặc treo vô hạn (`delay(Duration.INFINITE)`), toàn bộ bộ test suite có thể bị treo theo.

Với API `CoroutinesTimeout` (hỗ trợ cả JUnit 4 và JUnit 5), bộ kiểm thử sẽ **tự động cài đặt DebugProbes**, thiết lập hạn mức thời gian và **tự động dump toàn bộ danh sách coroutine đang hoạt động cùng stack trace chi tiết khi quá thời gian chờ (time out)**.

### 5.1 Sử dụng với JUnit 4
Để đặt thời gian chờ cho các bài test JUnit 4 và in stack trace coroutine khi vượt quá hạn mức, sử dụng JUnit `@Rule` với `CoroutinesTimeout`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.debug.junit4.CoroutinesTimeout
import org.junit.Rule
import org.junit.Test
import kotlin.time.Duration

@OptIn(ExperimentalCoroutinesApi::class)
class UserProfileTest {
    @get:Rule
    val timeout = CoroutinesTimeout.seconds(1) // Timeout sau 1 giây

    private suspend fun loadUserProfile() {
        withContext(Dispatchers.IO) {
            // Giả lập một tác vụ bị treo vô hạn
            delay(Duration.INFINITE)
        }
    }

    @Test
    fun loadsUserProfile() = runBlocking {
        val loadingJob = launch {
            loadUserProfile()
        }

        loadingJob.join() // Bài test sẽ bị fail sau 1s kèm Coroutine Dump chi tiết!
    }
}
```
Sau 1 giây, quy tắc sẽ báo cáo test bị quá hạn, in ra Coroutine dump và ném ngoại lệ `org.junit.runners.model.TestTimedOutException`.

### 5.2 Sử dụng với JUnit 5
Để áp dụng timeout cho toàn bộ các hàm test trong một class JUnit 5, gắn annotation `@CoroutinesTimeout` lên cấp độ class:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.debug.junit5.CoroutinesTimeout
import org.junit.jupiter.api.Test
import kotlin.time.Duration

@OptIn(ExperimentalCoroutinesApi::class)
// Thiết lập timeout 1 giây (1.000 ms) cho toàn bộ hàm test trong class
@CoroutinesTimeout(testTimeoutMs = 1_000)
class UserProfileTest {
    private suspend fun loadUserProfile() {
        withContext(Dispatchers.IO) {
            // Giả lập thao tác không bao giờ hoàn thành
            delay(Duration.INFINITE)
        }
    }

    @Test
    fun loadsUserProfile() = runBlocking {
        val loadingJob = launch {
            loadUserProfile()
        }

        // Chờ coroutine hoàn thành (sẽ không bao giờ xong)
        loadingJob.join()
    }
}
```
Sau 1 giây, bài test sẽ thất bại với ngoại lệ `kotlinx.coroutines.debug.junit5.CoroutinesTimeoutException` kèm theo Coroutines Dump chi tiết.

### 5.3 Xử lý Xung đột Tài nguyên của `kotlinx-coroutines-debug` trên Android
Debug agent **không được hỗ trợ trực tiếp trên môi trường Android runtime**.

Module `kotlinx-coroutines-debug` có các dependency bắc cầu (transitive) tới JNA, JNA Platform, Byte Buddy, và Byte Buddy Agent. Một số thư viện này chứa các file tài nguyên trùng đường dẫn. Khi Android Studio / Gradle thực hiện gộp tài nguyên dependency (merge resources), việc trùng lặp file có thể gây lỗi `DuplicateRelativeFileException` làm hỏng quá trình build.

Để giải quyết lỗi build này mà vẫn giữ dependency `kotlinx-coroutines-debug` trong dự án, hãy cấu hình khối `packaging` trong tệp `build.gradle.kts`:

```kotlin
// build.gradle.kts (Module: app)
android {
    packaging {
        resources {
            // Loại bỏ các file license trùng từ JNA và JNA Platform
            excludes += setOf(
                "META-INF/AL2.0",
                "META-INF/LGPL2.1",
            )

            // Loại bỏ file license ASM từ Byte Buddy
            excludes += "META-INF/licenses/ASM"

            // Chỉ giữ lại một bản sao cho mỗi file Byte Buddy Agent
            pickFirsts += setOf(
                "win32-x86-64/attach_hotspot_windows.dll",
                "win32-x86/attach_hotspot_windows.dll",
            )
        }
    }
}
```

---

## 6. Gỡ Lỗi Flow với IntelliJ IDEA Coroutine Debugger

Khi gỡ lỗi các đường ống Flow, IntelliJ IDEA và Android Studio cung cấp công cụ **Coroutines Tab** chuyên dụng trong cửa sổ Debug:

### 6.1 Quan sát Trạng thái RUNNING và SUSPENDED
Giả sử bạn có một Flow đơn giản với bên phát chậm và bên thu thập chậm:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i) // Đặt Breakpoint 1 tại đây
    }
}

fun main() = runBlocking {
    simple()
        .collect { value ->
            delay(300)
            println(value) // Đặt Breakpoint 2 tại đây
        }
}
```

1. Đặt Breakpoint tại dòng `emit(i)`.
2. Khởi chạy ở chế độ **Debug**.
3. Mở tab **Coroutines** trong cửa sổ **Debug tool window**:
   - **Frames tab**: Ngăn xếp hàm (Call stack).
   - **Variables tab**: Biến ngữ cảnh hiện tại.
   - **Coroutines tab**: Danh sách tất cả coroutine và trạng thái hiện thời của chúng.
4. Bấm **Resume Program**: Trình gỡ lỗi sẽ dừng tại điểm phát tiếp theo, cho phép bạn theo dõi giá trị luân chuyển qua từng chu kỳ.

### 6.2 Hiện tượng Biến bị Tối ưu Hóa ("was optimized out")
Khi debug các hàm `suspend`, đôi khi bạn sẽ thấy dòng chữ *"was optimized out"* bên cạnh tên một biến trong cửa sổ Variables.  
Điều này có nghĩa là trình biên dịch Kotlin đã tối ưu hóa vòng đời của biến (giảm thời gian sống trong bộ nhớ), khiến biến không còn tồn tại tại thời điểm suspend.

- **Cách vô hiệu hóa hành vi tối ưu này khi debug**: Thêm tùy chọn compiler `-Xdebug` trong cấu hình biên dịch.
- **Cảnh báo an toàn**:
  > [!CAUTION]
  > **TUYỆT ĐỐI KHÔNG sử dụng cờ `-Xdebug` trên môi trường Production**, vì việc giữ lại các tham chiếu biến quá lâu sau khi suspend có thể gây rò rỉ bộ nhớ (memory leaks) nghiêm trọng!

### 6.3 Gỡ Lỗi Flow Đa Coroutine với `.buffer()`
Khi thêm toán tử `.buffer()`, bên phát (emitter) và bên thu thập (collector) sẽ được tách thành **hai coroutine độc lập chạy đồng thời**:

```kotlin
fun main() = runBlocking<Unit> {
    simple()
        .buffer() // Tách collector sang một coroutine riêng
        .collect { value ->
            delay(300)
            println(value)
        }
}
```

Khi đặt breakpoint tại cả `emit(i)` và `println(value)`:
- Trong tab **Coroutines**, bạn sẽ thấy rõ **2 coroutine đồng thời**:
  - Khi một coroutine ở trạng thái **`RUNNING`**, coroutine kia ở trạng thái **`SUSPENDED`**.
  - Nhấp đúp chuột vào từng coroutine trong danh sách để xem call stack và trạng thái biến cục bộ riêng biệt của từng luồng phát và luồng thu thập.

---

## 7. Kiểm Thử Flow với Thư viện Turbine (Testing Flows with Turbine)

Đối với các đường ống luồng dữ liệu (Flow), việc kiểm thử bằng hàm `delay` hoặc gom `toList()` dễ gây ra các bài test chập chờn (flaky tests). Giải pháp chuẩn mực của cộng đồng Kotlin và Android là thư viện **Turbine** (`app.cash.turbine:turbine`):

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class FlowUnitTest {
    @Test
    fun testUserUpdates() = runTest {
        val userFlow = flowOf("Alice", "Bob", "Charlie")

        userFlow.test {
            assertEquals("Alice", awaitItem())
            assertEquals("Bob", awaitItem())
            assertEquals("Charlie", awaitItem())
            awaitComplete()
        }
    }
}
```

---

## 8. Bảng Tổng hợp Công cụ Gỡ lỗi

| Công cụ / Cờ cấu hình | Nền tảng | Công dụng cốt lõi |
| :--- | :--- | :--- |
| **`-Dkotlinx.coroutines.debug`** | JVM / Test | Bật tên coroutine (`@coroutine#X`), bật Stack Trace Recovery, chi phí cực thấp. |
| **`-Xdebug` (Compiler flag)** | Kotlin Compiler | Bỏ tối ưu hóa biến ("was optimized out") trong debugger. Cấm dùng trên production! |
| **Tab Coroutines (IntelliJ/AS)** | IDE Debugger | Theo dõi trực quan trạng thái `RUNNING` và `SUSPENDED` của từng coroutine riêng lẻ. |
| **`StackTraceRecoverable`** | Chuẩn Kotlin | Cho phép các ngoại lệ tùy biến tùy chỉnh bản sao phục hồi stack trace qua ranh giới coroutine. |
| **`DebugProbes`** | `kotlinx-coroutines-debug` | Dump danh sách coroutine, cây phân cấp Job/Scope, kiểm tra trạng thái RUNNING/SUSPENDED. |
| **`CoroutinesTimeout`** | JUnit 4/5 | Đặt thời gian timeout cho test và tự động in vết coroutine bị treo khi test thất bại. |
| **Turbine** | Đa nền tảng (KMP/Android) | Kiểm thử Flow tuần tự từng sự kiện (`awaitItem`, `awaitError`, `awaitComplete`). |

