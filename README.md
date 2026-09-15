# Android Knowledge Base

> Tài liệu học thuật chuyên sâu về **Android Modern Development** — Kotlin Coroutines/Flow và Jetpack Compose.  
> Chuẩn mực theo [Android Developers](https://developer.android.com), [Kotlin Documentation](https://kotlinlang.org) và [Google Architecture Guidelines](https://developer.android.com/topic/architecture).

---

> 📖 **Tài liệu chính thức được biên dịch 1:1:**
> - 👉 [**Kotlin Coroutines & Flow — JetBrains Official Guide**](official-coroutines-guide/README.md) (Trọn bộ 14 bài chuyên sâu từ Cài đặt, Basics, Flows, Channels, Debugging, Dự án thực hành Hands-on đến Sổ tay tra cứu & 20 câu hỏi phỏng vấn Senior).
> - 👉 [**Android Coroutines & Flow — Google Official Guide**](official-android-coroutines-flow/README.md) (Trọn bộ 6 bài chuyên sâu theo chuẩn Google Android Developers: Coroutines on Android, Best Practices, Flows, StateFlow/SharedFlow, Lifecycle-aware Collection, và Unit Testing với Turbine).

---

## Cấu trúc bài giảng

Mỗi bài giảng tuân theo cấu trúc **6 phần chuẩn**:

| # | Phần | Mô tả |
|---|------|--------|
| 1 | **Definition** | Định nghĩa & thuật ngữ chính xác theo official docs |
| 2 | **Under the Hood** | Cơ chế bên dưới — memory, threading, lifecycle |
| 3 | **Problem Statement** | Bài toán giải quyết, so sánh với cách cũ |
| 4 | **Best Practices** | Use cases thực tế, pitfalls, anti-patterns |
| 5 | **Implementation** | Code mẫu hoàn chỉnh Data → ViewModel → Compose UI |
| 6 | **FAQ & Troubleshooting** | Câu hỏi phỏng vấn Senior, lỗi runtime phổ biến |

---

## Module 1 — Kotlin Coroutines & Flow Foundation

| Bài | Chủ đề | Trạng thái |
|-----|--------|-----------|
| [01](module-1-coroutines-flow/01-coroutines-basics.md) | **Coroutines Basics — suspend, CoroutineScope, Dispatcher** | ✅ Hoàn chỉnh |
| [02](module-1-coroutines-flow/02-cold-flow-vs-hot-flow.md) | **Cold Flow vs Hot Flow — StateFlow, SharedFlow** | ✅ Hoàn chỉnh |
| [03](module-1-coroutines-flow/03-flow-operators.md) | **Flow Operators — map, flatMapLatest, combine, zip** | ✅ Hoàn chỉnh |
| [04](module-1-coroutines-flow/04-flow-lifecycle-android.md) | **Flow trong Android Lifecycle — collectAsStateWithLifecycle** | ✅ Hoàn chỉnh |
| [05](module-1-coroutines-flow/05-flow-exception-handling.md) | **Flow Exception Handling — catch, onCompletion, SupervisorJob** | ✅ Hoàn chỉnh |

## Module 2 — Architecture & ViewModel

| Bài | Chủ đề | Trạng thái |
|-----|--------|-----------|
| [06](module-2-architecture/06-viewmodel-flow-compose-udf.md) | **ViewModel + Flow + Compose — UDF end-to-end** | ✅ Hoàn chỉnh |
| [07](module-2-architecture/07-domain-layer-usecases.md) | **Domain Layer & Use Cases với Flow** | ✅ Hoàn chỉnh |
| [08](module-2-architecture/08-repository-pattern-flow.md) | **Repository Pattern với Flow — Offline-First** | ✅ Hoàn chỉnh |
| [09](module-2-architecture/09-paging3-flow.md) | **Paging 3 + Flow — Phân trang & Offline Cache** | ✅ Hoàn chỉnh |

## Module 3 — Jetpack Compose Foundation

| Bài | Chủ đề | Trạng thái |
|-----|--------|-----------|
| [10](module-3-compose-foundation/10-composable-functions.md) | **Composable Functions — Lifecycle, Slot API, Gap Buffer** | ✅ Hoàn chỉnh |
| [11](module-3-compose-foundation/11-compose-state.md) | **Compose State — remember, rememberSaveable, State Hoisting** | ✅ Hoàn chỉnh |
| [12](module-3-compose-foundation/12-recomposition-stability.md) | **Recomposition & Stability — @Stable, @Immutable, Metrics** | ✅ Hoàn chỉnh |
| [13](module-3-compose-foundation/13-side-effects.md) | **Side Effects — LaunchedEffect, DisposableEffect, SideEffect** | ✅ Hoàn chỉnh |

## Module 4 — Jetpack Compose Advanced

| Bài | Chủ đề | Trạng thái |
|-----|--------|-----------|
| [14](module-4-compose-advanced/14-navigation.md) | **Compose Navigation — Type-Safe 2.8+, Deep Links** | ✅ Hoàn chỉnh |
| [15](module-4-compose-advanced/15-animation.md) | **Compose Animation — Spring Physics, updateTransition** | ✅ Hoàn chỉnh |
| [16](module-4-compose-advanced/16-performance-lazy-layouts.md) | **Performance & Lazy Layouts — Keys, Baseline Profiles** | ✅ Hoàn chỉnh |
| [17](module-4-compose-advanced/17-compose-testing.md) | **Compose Testing — Semantics, UI Tests, Roborazzi** | ✅ Hoàn chỉnh |

## Module 5 — Advanced Patterns

| Bài | Chủ đề | Trạng thái |
|-----|--------|-----------|
| [18](module-5-advanced-patterns/18-mvi-architecture.md) | **MVI Architecture với Compose — Finite State Machine & Reducer** | ✅ Hoàn chỉnh |
| [19](module-5-advanced-patterns/19-hilt-flow-compose.md) | **Hilt + Flow + Compose — Dependency Injection End-to-End** | ✅ Hoàn chỉnh |
| [20](module-5-advanced-patterns/20-workmanager-flow.md) | **WorkManager + Flow — Reliable Background Work & Progress** | ✅ Hoàn chỉnh |
| [21](module-5-advanced-patterns/21-room-flow.md) | **Room + Flow — Reactive Database, WAL & Safe Migrations** | ✅ Hoàn chỉnh |

---

## Lộ trình học đề xuất

```
Beginner:  01 → 02 → 03 → 11 → 10 → 13
Mid-level: 04 → 05 → 06 → 07 → 08 → 12
Senior:    09 → 14 → 15 → 16 → 17 → 18 → 19 → 20 → 21
```

---

## Tech Stack & Versions

| Thư viện | Version |
|---------|---------|
| Kotlin | 2.0+ |
| Coroutines | 1.8+ |
| Compose BOM | 2024.06+ |
| Lifecycle | 2.8+ |
| ViewModel | 2.8+ |
| Hilt | 2.51+ |
| Turbine (test) | 1.1+ |
