# Jetpack Compose State — Google Official Guide (Bản dịch & Hệ thống hóa Chuẩn 1:1)

> **Tài liệu tham chiếu chính thức từ Google Android Developers:**
> - [State and Jetpack Compose — developer.android.com](https://developer.android.com/develop/ui/compose/state)
> - [Where to hoist state & State holders — developer.android.com](https://developer.android.com/develop/ui/compose/state-hoisting)
> - [Save UI state in Compose — developer.android.com](https://developer.android.com/develop/ui/compose/state-saving)
> - [Jetpack Compose phases & Deferring state reads — developer.android.com](https://developer.android.com/develop/ui/compose/phases)
> - [Side-effects in Compose & derivedStateOf — developer.android.com](https://developer.android.com/develop/ui/compose/side-effects)
> - [State holders and UI State — developer.android.com](https://developer.android.com/develop/ui/compose/state-holders)
>
> **Phiên bản áp dụng:** Kotlin 2.0+, Jetpack Compose BOM 2024.06+, Compose Compiler 2.0+, AndroidX Lifecycle 2.8+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Giới thiệu Bộ tài liệu

Trong kiến trúc khai báo (Declarative UI) của Jetpack Compose, **State (Trạng thái)** là trái tim điều khiển mọi hành vi hiển thị: **Giao diện người dùng là hàm số thuần túy của State**:

$$\text{UI} = f(\text{State})$$

Bộ tài liệu này được biên soạn và hệ thống hoá theo chuẩn cấu trúc và nội dung từ **Google Android Developers**, cung cấp góc nhìn từ nguyên lý cơ bản đến cơ chế hoạt động tầng thấp (Under the Hood):

- **Bám sát triết lý Unidirectional Data Flow (UDF):** Trạng thái truyền xuống (State flows down), sự kiện phát ra ngược lên (Events flow up) giúp UI tách rời hoàn toàn khỏi logic nghiệp vụ.
- **Hệ thống phân cấp State Holders 3 tầng:** Làm rõ ranh giới khi nào quản lý State trong Composable, khi nào đóng gói vào Plain State Holder class (scoping to Composition), và khi nào đẩy lên AAC ViewModel (scoping to Screen/NavBackStackEntry).
- **Chiến lược bảo toàn dữ liệu đa tầng:** Xử lý triệt để 3 cấp độ mất trạng thái (Recomposition, Configuration Change, Process Death) với sự phối hợp giữa `rememberSaveable` và `SavedStateHandle`.
- **Tối ưu hóa hiệu năng 60/120 FPS:** Hiểu sâu 3 pha của Compose (**Composition $\rightarrow$ Layout $\rightarrow$ Draw**) và kỹ thuật hoãn đọc State (Defer State Reads) qua lambdas để bỏ qua Recomposition không cần thiết.
- **Đi sâu vào cơ chế lõi Runtime:** Phân tích Snapshot State System dựa trên kiến trúc MVCC (Multi-Version Concurrency Control), Read/Write tracking, cùng quy tắc Stability và Strong Skipping Mode trong Kotlin 2.0+.
- **Kiểm thử State & State Holders:** Hướng dẫn Unit Test cho State Holder độc lập và UI Test với `createComposeRule` cùng `StateRestorationTester`.

---

## 2. Mục Lục Chi Tiết & Bảng Đối Chiếu

Mỗi bài học đều tuân thủ nghiêm ngặt cấu trúc học thuật **6 phần chuẩn**:  
`1. Definition` $\rightarrow$ `2. Under the Hood` $\rightarrow$ `3. Problem Statement & Architecture` $\rightarrow$ `4. Best Practices & Anti-patterns` $\rightarrow$ `5. Implementation` $\rightarrow$ `6. FAQ & Troubleshooting`.

| STT | Bài học (Tiếng Việt) | Nguồn tài liệu gốc (Google Developers) | Trọng tâm kiến thức |
| :---: | :--- | :--- | :--- |
| **01** | [**State và Jetpack Compose**](01-state-and-jetpack-compose.md) | [State and Jetpack Compose](https://developer.android.com/develop/ui/compose/state) | Bản chất State & Event, mô hình $UI = f(State)$, `mutableStateOf`, `remember`, cú pháp `by`, Stateful vs Stateless, State Hoisting, Single Source of Truth (SSOT). |
| **02** | [**Phân Cấp State & State Holders**](02-where-to-hoist-state-and-state-holders.md) | [Where to hoist state](https://developer.android.com/develop/ui/compose/state-hoisting) | Phân biệt UI Element State vs Screen UI State, UI Logic vs Business Logic, Decision Tree lựa chọn Composable vs Plain State Holder vs `ViewModel`. |
| **03** | [**Lưu Trữ & Khôi Phục UI State**](03-saving-ui-state.md) | [Save UI state in Compose](https://developer.android.com/develop/ui/compose/state-saving) | 3 cấp độ mất state, `rememberSaveable`, `SavedStateRegistry`, `@Parcelize`, tự viết `listSaver` / `mapSaver`, tích hợp `SavedStateHandle` trong ViewModel. |
| **04** | [**3 Pha Compose & Kỹ Thuật Hoãn Đọc State**](04-compose-phases-and-deferring-state-reads.md) | [Jetpack Compose phases](https://developer.android.com/develop/ui/compose/phases) | 3 pha (Composition $\rightarrow$ Layout $\rightarrow$ Draw), ranh giới đọc state, hoãn đọc State bằng Lambdas (`Modifier.offset { }`, `graphicsLayer { }`), tối ưu mượt 120 FPS. |
| **05** | [**derivedStateOf, Side Effects & snapshotFlow**](05-derived-state-and-side-effects.md) | [Side-effects in Compose](https://developer.android.com/develop/ui/compose/side-effects) | `derivedStateOf` chống Over-recomposition khi state đổi tần suất cao, phân biệt với `remember(key)`, chuyển đổi State sang Flow bằng `snapshotFlow`. |
| **06** | [**Snapshot State System & Stability**](06-snapshot-state-system-and-stability.md) | [Compose Architecture & Compiler](https://developer.android.com/develop/ui/compose/mental-model) | Kiến trúc MVCC Snapshot System, Read/Write tracking, `SnapshotMutationPolicy`, cơ chế Stability (`@Stable`, `@Immutable`), Strong Skipping Mode trong Kotlin 2.0+. |
| **07** | [**Thực Hành Kiến Trúc & Kiểm Thử State**](07-testing-and-production-state-patterns.md) | Testing & Production Patterns | Xây dựng Case Study phức tạp kết hợp cả 3 tầng State Holder, Unit Test Plain State Holder độc lập, UI Test với `createComposeRule` và `StateRestorationTester`. |

---

## 3. Bản Đồ Kiến Trúc State & State Holders trong Jetpack Compose

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                   SCREEN LAYER                                    │
│                                                                                   │
│  [Screen UI State] (Business data: Danh sách sản phẩm, Auth status, User profile)  │
│  Nguồn phát sinh: Mạng, Database, Business rules                                  │
│  State Holder: SCREEN-LEVEL STATE HOLDER (ViewModel)                              │
│  - Vòng đời: Scoped to NavBackStackEntry / ViewModelStore                         │
│  - Sống sót qua: Xoay màn hình (Rotate), Config changes                           │
│  - Chống Process Death: SavedStateHandle                                          │
└─────────────────────────────────────────┬─────────────────────────────────────────┘
                                          │ StateFlow<UiState> / Events: onIntent()
                                          ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                           COMPOSITION / UI ELEMENT LAYER                          │
│                                                                                   │
│  [UI Element State] (Trạng thái giao diện: Drawer mở/đóng, Scroll offset, Input)  │
│  Nguồn phát sinh: Tương tác người dùng trực tiếp trên widget                      │
│                                                                                   │
│  ┌─────────────────────────────────────┐   ┌───────────────────────────────────┐  │
│  │     Simple UI Element State         │   │     Complex UI Element State      │  │
│  │     Quản lý ngay tại Composable     │   │     PLAIN STATE HOLDER CLASS      │  │
│  │     - remember { mutableStateOf }   │   │     - class MyAppState(...)       │  │
│  │     - rememberSaveable              │   │     - @Composable rememberState() │  │
│  └─────────────────────────────────────┘   └───────────────────────────────────┘  │
│  Vòng đời: Phụ thuộc vào Composition Tree (mất khi Composable rời khỏi tree)       │
│  Chống Config Change / Process Death: rememberSaveable (SavedStateRegistry)       │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Thiết Lập Gradle Dependencies Chuẩn Mực

Thêm các dependencies sau vào file `build.gradle.kts` (Module: `app`):

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler) // Kotlin 2.0+ tích hợp Compose Compiler Plugin
    alias(libs.plugins.kotlin.parcelize) // Hỗ trợ @Parcelize cho rememberSaveable
}

dependencies {
    // 1. Compose BOM (Bill of Materials) - Quản lý phiên bản thống nhất
    val composeBom = platform("androidx.compose:compose-bom:2024.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)

    // 2. Compose UI, Foundation & Material 3
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.foundation:foundation")

    // 3. AndroidX Lifecycle & ViewModel cho Compose
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.4") // collectAsStateWithLifecycle

    // 4. SavedStateHandle & KTX
    implementation("androidx.lifecycle:lifecycle-viewmodel-savedstate:2.8.4")

    // 5. Compose Debug Tools & Testing
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // 6. Unit Test & UI Test cho Compose State
    testImplementation("junit:junit:4.13.2")
    testImplementation("com.google.truth:truth:1.4.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
}
```
