# Kotlin Coroutines & Flow — Official Guide (Bản dịch chuẩn 1:1)

> **Tài liệu tham chiếu:** [Kotlin Coroutines Guide — kotlinlang.org](https://kotlinlang.org/docs/coroutines-guide.html)  
> **Phiên bản áp dụng:** Kotlin 2.0+, `kotlinx.coroutines` 1.9+ / 1.11+  
> **Ngôn ngữ:** Tiếng Việt (kèm thuật ngữ kỹ thuật chuẩn quốc tế)

---

## Giới thiệu Series

Bộ tài liệu này được biên soạn theo **chuẩn dịch thuật ngữ và cấu trúc 1:1** trực tiếp từ tài liệu chính thức của JetBrains về Kotlin Coroutines và Flow. Mỗi bài học tương ứng chính xác với một chương trong cẩm nang chính thức, bảo toàn toàn bộ:
- Mạch tư duy lý thuyết và thứ tự tiếp cận từ cơ bản đến chuyên sâu.
- 100% các đoạn mã mẫu (code snippets) có thể chạy độc lập (runnable).
- Kết quả đầu ra chuẩn (Standard Console Output).
- Các hộp lưu ý đặc biệt (`Note`, `Tip`, `Warning`) từ đội ngũ thiết kế ngôn ngữ Kotlin.
- Bổ sung các góc nhìn phân tích cơ chế JVM State Machine, CPS (Continuation-Passing Style) và Threading bên dưới.

---

## Mục lục Toàn diện & Bảng đối chiếu

| STT | Bài học (Tiếng Việt) | Chương tài liệu gốc (kotlinlang.org) | Trạng thái |
| :---: | :--- | :--- | :---: |
| **00** | [**Tổng quan & Cài đặt Môi trường**](00-overview-and-setup.md) | [Coroutines Guide Overview](https://kotlinlang.org/docs/coroutines-guide.html) | ✅ Hoàn thành |
| **01** | [**Cơ bản về Coroutines (Coroutines Basics)**](01-coroutines-basics.md) | [Coroutines basics](https://kotlinlang.org/docs/coroutines-basics.html) | ✅ Hoàn thành |
| **02** | [**Hủy bỏ & Giới hạn Thời gian (Cancellation & Timeouts)**](02-cancellation-and-timeouts.md) | [Cancellation and timeouts](https://kotlinlang.org/docs/coroutines-cancellation.html) | ✅ Hoàn thành |
| **03** | [**Kết hợp các Hàm Suspend (Composing Suspending Functions)**](03-composing-suspending-functions.md) | [Composing suspending functions](https://kotlinlang.org/docs/composing-suspending-functions.html) | ✅ Hoàn thành |
| **04** | [**Ngữ cảnh Coroutine & Bộ điều phối (Context & Dispatchers)**](04-coroutine-context-and-dispatchers.md) | [Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html) | ✅ Hoàn thành |
| **05** | [**Asynchronous Flow: Khái niệm Cốt lõi & Vận hành**](05-flows-core.md) | [Flows](https://kotlinlang.org/docs/coroutines-flow.html) | ✅ Hoàn thành |
| **06** | [**Các Toán tử trong Flow (Flow Operators Deep Dive)**](06-flow-operators.md) | [Flow operators](https://kotlinlang.org/docs/coroutines-flow-operators.html) | ✅ Hoàn thành |
| **07** | [**Kênh Giao tiếp Coroutine (Channels & Pipelines)**](07-channels.md) | [Channels](https://kotlinlang.org/docs/channels.html) | ✅ Hoàn thành |
| **08** | [**Xử lý Ngoại lệ trong Coroutine (Exception Handling & Supervision)**](08-exception-handling.md) | [Coroutine exceptions handling](https://kotlinlang.org/docs/exception-handling.html) | ✅ Hoàn thành |
| **09** | [**Trạng thái Khả biến Chia sẻ & Đồng thời (Shared Mutable State & Mutex)**](09-shared-mutable-state-and-concurrency.md) | [Shared mutable state and concurrency](https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html) | ✅ Hoàn thành |
| **10** | [**Biểu thức Select (Select Expression — Experimental)**](10-select-expression.md) | [Select expression (experimental)](https://kotlinlang.org/docs/select-expression.html) | ✅ Hoàn thành |
| **11** | [**Thực hành & Debugging Coroutines/Flow với IntelliJ IDEA**](11-debugging-coroutines-and-flow.md) | [Debug coroutines](https://kotlinlang.org/docs/debug-coroutines-with-idea.html) & [Debug Flow](https://kotlinlang.org/docs/debug-flow-with-idea.html) | ✅ Hoàn thành |
| **12** | [**Dự án Thực hành: GitHub Contributors & Channels**](12-hands-on-github-contributors.md) | [Coroutines & channels tutorial](https://kotlinlang.org/docs/coroutines-and-channels.html) | ✅ Hoàn thành |
| **13** | [**Sổ Tay Tra Cứu & 20 Câu Hỏi Phỏng Vấn Hóc Búa**](13-cheatsheet-and-interview-guide.md) | JetBrains Official & Senior Interview Guide | ✅ Hoàn thành |

---

## Lộ trình Học tập Khuyến nghị

```
                  ┌───────────────────────────────┐
                  │ 00. Tổng quan & Thiết lập     │
                  └───────────────┬───────────────┘
                                  ▼
                  ┌───────────────────────────────┐
                  │ 01. Coroutines Basics         │
                  └───────┬───────────────┬───────┘
                          │               │
            ┌─────────────▼─────┐   ┌─────▼─────────────┐
            │ 02. Cancellation  │   │ 03. Composing     │
            │     & Timeouts    │   │     Functions     │
            └─────────────┬─────┘   └─────┬─────────────┘
                          │               │
                          └───────┬───────┘
                                  ▼
                  ┌───────────────────────────────┐
                  │ 04. Context & Dispatchers     │
                  └───────┬───────────────┬───────┘
                          │               │
        ┌─────────────────▼─────┐   ┌─────▼─────────────────┐
        │ 05. Flows Core        │   │ 07. Channels          │
        │ 06. Flow Operators    │   │ 08. Exceptions        │
        └─────────────────┬─────┘   │ 09. Shared State      │
                          │         │ 10. Select Expression │
                          │         └─────┬─────────────────┘
                          └───────┬───────┘
                                  ▼
                  ┌───────────────────────────────┐
                  │ 11. Debugging & Verification  │
                  └───────────────────────────────┘
```

---

## Hướng dẫn Thiết lập Dependency

### Gradle (Kotlin DSL - `build.gradle.kts`)
```kotlin
dependencies {
    // Thư viện lõi Coroutines đa nền tảng (JVM, Android, Native, JS)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")

    // Hỗ trợ Android Main Dispatcher (nếu phát triển ứng dụng Android)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0")

    // Thư viện kiểm thử Coroutines
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")
    testImplementation("app.cash.turbine:turbine:1.2.0")
}
```

### Gradle (Groovy DSL - `build.gradle`)
```groovy
dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0'
    testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0'
}
```

### Maven (`pom.xml`)
```xml
<dependency>
    <groupId>org.jetbrains.kotlinx</groupId>
    <artifactId>kotlinx-coroutines-core</artifactId>
    <version>1.11.0</version>
</dependency>
```
