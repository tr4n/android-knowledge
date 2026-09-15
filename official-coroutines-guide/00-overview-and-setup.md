# Bài 00 — Tổng quan về Kotlin Coroutines & Cài đặt Môi trường (Coroutines Guide Overview)

> **Tài liệu gốc:** [Coroutines guide — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-guide.html)  
> **Thư viện chính thức:** `org.jetbrains.kotlinx:kotlinx-coroutines-core`  
> **Phiên bản:** Kotlin 2.0+, `kotlinx.coroutines` 1.11.0  

---

## 1. Triết lý Thiết kế của Kotlin Coroutines

Kotlin chỉ cung cấp các API cấp thấp tối thiểu trong thư viện chuẩn (`kotlin-stdlib`) nhằm tạo điều kiện cho các thư viện khác xây dựng và tận dụng coroutines.

Khác biệt hoàn toàn với nhiều ngôn ngữ lập trình khác có tính năng tương tự:
1. **`async` và `await` KHÔNG PHẢI là từ khóa trong Kotlin**:
   Chúng thậm chí không nằm trong thư viện chuẩn của ngôn ngữ. Trong Kotlin, `async` chỉ là một hàm thông thường (coroutine builder) do thư viện `kotlinx.coroutines` cung cấp, và `await()` là một phương thức của interface `Deferred<T>`.
2. **Khái niệm Hàm tạm dừng (Suspending Functions)**:
   Khái niệm **suspending function** mang lại một tầng trừu tượng an toàn hơn, trực quan hơn và ít phát sinh lỗi hơn rất nhiều cho các thao tác bất đồng bộ so với mô hình Futures và Promises truyền thống. Bạn có thể viết mã bất đồng bộ theo phong cách tuần tự tuyến tính (sequential style) mà không lo ngại hiện tượng "Callback Hell" hay phân mảnh luồng điều khiển.

---

## 2. Thư viện `kotlinx.coroutines`

`kotlinx.coroutines` là một thư viện phong phú dành cho coroutines do chính **JetBrains** phát triển và duy trì. Thư viện này chứa một loạt các nguyên ngữ đồng thời cấp cao (high-level coroutine-enabled primitives) mà cẩm nang này sẽ đề cập chi tiết, bao gồm:
- Các hàm khởi tạo coroutine: `launch`, `async`, `runBlocking`.
- Quản lý phạm vi và vòng đời: `CoroutineScope`, `coroutineScope`, `supervisorScope`, `Job`.
- Luồng dữ liệu bất đồng bộ: `Flow`, `StateFlow`, `SharedFlow`.
- Giao tiếp giữa các tiến trình đồng thời: `Channel`, `produce`, `select`.
- Đồng bộ hóa không khóa luồng: `Mutex`, `Semaphore`.

Đây là cẩm nang toàn diện về các tính năng cốt lõi của `kotlinx.coroutines` đi kèm một loạt ví dụ mã nguồn thực hành, được phân chia thành các chủ đề chuyên biệt.

---

## 3. Cài đặt Dependency trong Dự án

Để sử dụng coroutines cũng như thực thi toàn bộ các ví dụ trong loạt bài hướng dẫn này, bạn cần khai báo dependency của module `kotlinx-coroutines-core` vào dự án:

### 3.1 Gradle (Kotlin DSL — `build.gradle.kts`)

```kotlin
// build.gradle.kts
repositories {
    mavenCentral()
}

dependencies {
    // Thư viện lõi Coroutines (đa nền tảng: JVM, Android, Native, JS, Wasm)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")

    // Hỗ trợ Dispatchers.Main trên nền tảng Android (nếu có)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0")

    // Thư viện kiểm thử Coroutines
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")
    testImplementation("app.cash.turbine:turbine:1.2.0")
}
```

### 3.2 Gradle (Groovy DSL — `build.gradle`)

```groovy
// build.gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0'
    testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0'
}
```

### 3.3 Maven (`pom.xml`)

```xml
<!-- pom.xml -->
<project>
    <dependencies>
        <dependency>
            <groupId>org.jetbrains.kotlinx</groupId>
            <artifactId>kotlinx-coroutines-core</artifactId>
            <version>1.11.0</version>
        </dependency>
    </dependencies>
</project>
```

---

## 4. Mục lục Cẩm nang Chính thức (Table of Contents)

Dưới đây là cây mục lục đầy đủ bám sát 100% cấu trúc của cẩm nang chính thức:

1. [**Coroutines basics (Cơ bản về Coroutines)**](01-coroutines-basics.md): Khái niệm coroutine, hàm suspend, structured concurrency, builders (`launch`, `async`, `runBlocking`), dispatchers, và so sánh bộ nhớ với JVM threads.
2. [**Cancellation and timeouts (Hủy bỏ và Giới hạn thời gian)**](02-cancellation-and-timeouts.md): Hủy coroutine qua Job, tính hợp tác (cooperative cancellation), `yield()`, giải phóng tài nguyên với `try/finally`, khối không thể hủy `NonCancellable`, `runInterruptible`, và xử lý timeout.
3. [**Flows (Asynchronous Flow)**](05-flows-core.md): Luồng bất đồng bộ, Emitter - Operator - Collector, Cold flows vs Hot flows (`StateFlow`, `SharedFlow`), nguyên lý Exception Transparency, phát dữ liệu với `channelFlow`.
4. [**Composing suspending functions (Kết hợp các hàm Suspend)**](03-composing-suspending-functions.md): Thực thi tuần tự mặc định, tính toán đồng thời với `async`, khởi chạy lười `CoroutineStart.LAZY`, phân tích lý do bài trừ async-style functions.
5. [**Coroutine context and dispatchers (Ngữ cảnh Coroutine và Bộ điều phối)**](04-coroutine-context-and-dispatchers.md): Vai trò của `CoroutineContext`, các bộ điều phối luồng `Dispatchers`, debug với logging `-Dkotlinx.coroutines.debug`, quan hệ cha-con, trách nhiệm bảo bọc của coroutine cha, và dữ liệu Thread-local.
6. [**Channels (Kênh giao tiếp)**](07-channels.md): Trao đổi luồng dữ liệu giữa các coroutines qua Channel (`send`/`receive`), pipelines, sàng số nguyên tố, fan-out, fan-in, các kiểu dung lượng đệm, tính công bằng (fairness).
7. [**Coroutine exceptions handling (Xử lý ngoại lệ trong Coroutine)**](08-exception-handling.md): Cơ chế lan truyền lỗi (`launch` vs `async`), `CoroutineExceptionHandler`, hủy lan truyền hai chiều, tổng hợp ngoại lệ `suppressedExceptions`, cô lập lỗi với `SupervisorJob` và `supervisorScope`.
8. [**Shared mutable state and concurrency (Trạng thái khả biến chia sẻ và Đồng thời)**](09-shared-mutable-state-and-concurrency.md): Tranh chấp dữ liệu (race condition), tại sao `@Volatile` không đủ, biến nguyên tử atomic, giới hạn thread hạt mịn vs hạt thô, loại trừ tương hỗ với `Mutex`.
9. [**Select expression (Biểu thức Select — Experimental)**](10-select-expression.md): Lắng nghe đồng thời nhiều sự kiện bất đồng bộ và phản hồi trên sự kiện đầu tiên sẵn sàng (`onReceiveCatching`, `onSend`, `onAwait`).
10. [**Tutorial: Debug coroutines using IntelliJ IDEA & Debug Flow**](11-debugging-coroutines-and-flow.md): Gỡ lỗi trực quan trên IntelliJ IDEA Coroutine Debugger, theo dõi trạng thái Suspended, Coroutine Dump, đặt breakpoint trên Flow pipeline, và Unit test với Turbine.

---

## 5. Tài liệu Tham khảo Bổ sung (Additional References)

Dưới đây là các tài liệu tham khảo chính thức bổ trợ được JetBrains khuyến nghị:

- [**Guide to UI programming with coroutines**](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md): Cẩm nang lập trình giao diện người dùng (UI) với Coroutines.
- [**Coroutines design document (KEEP)**](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md): Tài liệu thiết kế chi tiết kỹ thuật của Kotlin Coroutines (Kotlin Evolution and Enhancement Process).
- [**Full kotlinx.coroutines API reference**](https://kotlinlang.org/api/kotlinx.coroutines/): Toàn bộ tài liệu tra cứu API chi tiết của thư viện `kotlinx.coroutines`.
- [**Best practices for coroutines in Android**](https://developer.android.com/kotlin/coroutines/coroutines-best-practices): Các nguyên tắc thiết kế và thực hành tốt nhất khi dùng Coroutines trên nền tảng Android.
- [**Additional Android resources for Kotlin coroutines and flow**](https://developer.android.com/kotlin/coroutines/additional-resources): Các tài nguyên chuyên sâu khác dành cho Android Developers.

---

## 6. Bảng Thuật ngữ Đối chiếu Quốc tế

| Thuật ngữ Tiếng Anh | Bản dịch Tiếng Việt | Ghi chú & Ý nghĩa kỹ thuật |
| :--- | :--- | :--- |
| **Coroutine** | Coroutine (tiến trình đồng thời mức người dùng) | Tác vụ có thể tạm dừng và tiếp tục mà không chặn OS Thread |
| **Suspending function** | Hàm tạm dừng | Hàm có từ khóa `suspend`, có khả năng nhường Thread |
| **Suspension point** | Điểm tạm dừng | Vị trí bên trong hàm suspend mà coroutine có thể dừng lại |
| **Structured concurrency** | Tính đồng thời có cấu trúc | Nguyên lý quản lý vòng đời coroutine theo cấp bậc cha - con |
| **Coroutine builder** | Hàm khởi tạo coroutine | Các hàm như `launch`, `async`, `runBlocking` |
| **Dispatcher** | Bộ điều phối luồng | Thành phần phân phối coroutine tới Thread Pool thực thi |
| **Cold stream / Cold flow** | Luồng dữ liệu nguội (lười) | Chỉ kích hoạt tính toán khi có Collector lắng nghe |
| **Hot stream / Hot flow** | Luồng dữ liệu nóng (chủ động) | Tự phát dữ liệu độc lập với việc có Collector hay không |
| **Backpressure** | Áp lực ngược | Hiện tượng bên phát dữ liệu nhanh hơn bên tiêu thụ xử lý |
| **Conflation** | Gộp / Bỏ qua giá trị cũ | Chỉ giữ lại giá trị mới nhất khi bên tiêu thụ bị chậm |
| **Thread confinement** | Giới hạn luồng | Chiến lược ép toàn bộ thao tác truy cập dữ liệu vào một Thread duy nhất |
| **Mutual exclusion (`Mutex`)** | Loại trừ tương hỗ | Khóa bảo vệ tài nguyên chia sẻ nhưng không làm block Thread |
