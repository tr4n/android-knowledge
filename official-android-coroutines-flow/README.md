# Android Coroutines & Flow — Google Official Guide (Bản dịch & Hệ thống hóa Chuẩn 1:1)

> **Tài liệu tham chiếu chính thức:**
> - [Kotlin coroutines on Android — developer.android.com](https://developer.android.com/kotlin/coroutines?hl=vi)
> - [Best practices for coroutines in Android — developer.android.com](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
> - [Kotlin flows on Android — developer.android.com](https://developer.android.com/kotlin/flow?hl=vi)
> - [StateFlow and SharedFlow — developer.android.com](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
>
> **Phiên bản áp dụng:** Kotlin 2.0+, `kotlinx.coroutines` 1.9+ / 1.11+, AndroidX Lifecycle 2.8+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Giới thiệu Bộ tài liệu

Bộ tài liệu này được biên soạn và hệ thống hoá theo chuẩn cấu trúc và nội dung từ **Google Android Developers**, tập trung chuyên biệt vào cách thức vận hành của Coroutines và Asynchronous Flow trong hệ sinh thái Android:

- **Bám sát kiến trúc hiện đại (Modern Android Development - MAD):** Tích hợp chặt chẽ với Jetpack Lifecycle, ViewModel, Jetpack Compose, Room và Retrofit.
- **Bảo toàn 100% mã nguồn mẫu chính thức:** Các ví dụ thực tế được Google thiết kế để giải quyết vấn đề quản lý thread, xử lý tác vụ mạng chạy lâu (long-running tasks) và tránh đơ giật giao diện (ANR - Application Not Responding).
- **Phân tích sâu cơ chế Main-Safety & Structured Concurrency:** Hướng dẫn cách phân chia trách nhiệm giữa các tầng (UI Layer, Domain Layer, Data Layer) sao cho an toàn tuyệt đối với luồng chính (Main thread).
- **Tổng hợp đầy đủ các Best Practices độc quyền từ Google Engineering:** Bao gồm các quy tắc Dependency Injection cho Dispatcher, hủy bỏ coroutine có phối hợp (cooperative cancellation) và mô hình StateFlow/SharedFlow xử lý sự kiện giao diện.

---

## 2. Mục lục Chi tiết & Bảng đối chiếu

| STT | Bài học (Tiếng Việt) | Nguồn tài liệu gốc (Google Developers) | Trọng tâm kiến thức |
| :---: | :--- | :--- | :--- |
| **01** | [**Kotlin Coroutines trên Android**](01-kotlin-coroutines-on-android.md) | [Kotlin coroutines on Android](https://developer.android.com/kotlin/coroutines?hl=vi) | 4 tính năng cốt lõi, Main-safety, `Dispatchers`, `viewModelScope`, kiến trúc đăng nhập mẫu. |
| **02** | [**Thực hành Tốt nhất cho Coroutines (Best Practices)**](02-coroutines-best-practices.md) | [Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices) | 9 quy tắc vàng từ Google: Inject Dispatchers, Suspend Main-safe, Không expose Mutable types, Cooperative cancellation. |
| **03** | [**Kotlin Flows trên Android**](03-kotlin-flows-on-android.md) | [Kotlin flows on Android](https://developer.android.com/kotlin/flow?hl=vi) | Asynchronous Stream, Producer-Intermediary-Consumer, Operators, `catch` & Exception Transparency, `flowOn`. |
| **04** | [**StateFlow & SharedFlow**](04-stateflow-and-sharedflow.md) | [StateFlow & SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow) | UI State Holder, Cold vs Hot Flow, `stateIn`, `shareIn`, chiến lược `WhileSubscribed(5000)`. |
| **05** | [**Thu thập Flow theo Vòng đời Android (Lifecycle-aware Collection)**](05-lifecycle-aware-flow-collection.md) | Google Architecture & Lifecycle Guide | `repeatOnLifecycle`, `flowWithLifecycle`, `collectAsStateWithLifecycle` trong Jetpack Compose, chống rò rỉ tài nguyên. |

---

## 3. Bản đồ Kiến trúc Coroutines & Flow trong Android

```
┌────────────────────────────────────────────────────────────────────────┐
│ UI LAYER (Jetpack Compose / Activity / Fragment)                       │
│                                                                        │
│  - Thu thập an toàn: collectAsStateWithLifecycle() / repeatOnLifecycle │
│  - Phát tín hiệu tương tác: onClick -> viewModel.doAction()           │
└───────────────────────────────────▲────────────────────────────────────┘
                                    │ StateFlow<UiState> / SharedFlow<UiEvent>
┌───────────────────────────────────┴────────────────────────────────────┐
│ PRESENTATION LAYER (ViewModel)                                         │
│                                                                        │
│  - Khởi tạo Coroutine: viewModelScope.launch                           │
│  - Quản lý trạng thái: stateIn(WhileSubscribed(5000))                  │
│  - Gọi Suspend functions từ Repository                                 │
└───────────────────────────────────▲────────────────────────────────────┘
                                    │ Suspend functions & Flow<Data>
┌───────────────────────────────────┴────────────────────────────────────┐
│ DATA / REPOSITORY LAYER (Repository & Data Sources)                   │
│                                                                        │
│  - Đảm bảo Main-safety: withContext(ioDispatcher)                      │
│  - Xuất dữ liệu luồng: Room Database (Flow<T>), Retrofit API (suspend) │
│  - Không leak Scope, xử lý Business Exceptions                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Thiết lập Gradle Dependencies chuẩn Android

Thêm các dependencies sau vào file `build.gradle.kts` (Module: `app`):

```kotlin
dependencies {
    // 1. Kotlin Coroutines Core & Android
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0")

    // 2. AndroidX Lifecycle (ViewModelScope, LifecycleScope, repeatOnLifecycle)
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")

    // 3. Lifecycle-aware Flow Collection cho Jetpack Compose
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.7")

    // 4. Room Database với Coroutines & Flow
    implementation("androidx.room:room-ktx:2.6.1")

    // 5. Kiểm thử Coroutines & Flow (Unit Test)
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")
    testImplementation("app.cash.turbine:turbine:1.2.0")
}
```
