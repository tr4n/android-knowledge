# Bài 06 — Compose Snapshot State System & Stability: Cơ Chế MVCC & Strong Skipping Mode

> **Tài liệu tham chiếu chính thức:** [Thinking in Compose & Compose Compiler Metrics — Android Developers](https://developer.android.com/develop/ui/compose/mental-model)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Compose Compiler 2.0+, Compose BOM 2024.06+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Để Compose có thể xử lý việc đọc và ghi State một cách mượt mà, đồng thời cho phép tính toán Recomposition trên nhiều luồng nền mà không bao giờ gây ra hiện tượng xung đột dữ liệu (Race Condition) hay giật lag giao diện, Compose Runtime được xây dựng trên hai nền tảng kiến trúc cốt lõi:

- **Snapshot State System (Hệ thống trạng thái Snapshot):** Một kiến trúc quản lý đồng thời lấy cảm hứng trực tiếp từ kỹ thuật **MVCC (Multi-Version Concurrency Control)** trong các hệ quản trị cơ sở dữ liệu quan hệ (như PostgreSQL, Oracle). Mỗi luồng hoặc mỗi chu kỳ Composition làm việc trên một "bản chụp" (Snapshot) cô lập của dữ liệu.
- **Stability System (Hệ thống tính ổn định):** Hệ thống phân tích tĩnh do Compose Compiler thực hiện để xác định xem một kiểu dữ liệu có **bất biến (Immutable)** hoặc **ổn định (Stable)** hay không.
- **Composable Skipping (Bỏ qua Recomposition):** Khả năng của Compose Compiler cho phép bỏ qua hoàn toàn việc gọi lại một hàm `@Composable` nếu tất cả các tham số truyền vào không hề thay đổi giá trị.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế MVCC Snapshot System: Read/Write Tracking

Trong Compose, mỗi một đối tượng State (như `MutableState<T>`) kế thừa từ interface nội bộ `StateObject`. Thay vì lưu giá trị trực tiếp vào một biến duy nhất, nó lưu trữ một danh sách liên kết các bản ghi phiên bản gọi là **`StateRecord`**.

```
                           CƠ CHẾ HOẠT ĐỘNG CỦA SNAPSHOT MVCC
                           
    GlobalSnapshot (Phiên bản gốc) ────────────► StateRecord (v1: value = 10)
              │
              ├───────► takeMutableSnapshot()
              ▼
    Snapshot Worker Thread (Luồng nền):
    - Đọc State: readable(this, snapshot) ──► Đọc bản chụp v1
    - Ghi State: writable(this, snapshot) ──► Tạo bản ghi mới: StateRecord (v2: value = 20)
              │
              ▼ snapshot.apply()
    ĐỐI CHIẾU XUNG ĐỘT (Conflict Detection) & HỢP NHẤT VÀO GLOBALS NAPSHOT:
    - Báo cáo Snapshot.sendApplyNotifications()
    - Kích hoạt Recomposition các Composable đã đọc biến này ở v1!
```

1. **`readable(this, snapshot)`:** Khi một hàm `@Composable` đọc giá trị `state.value`, Compose Runtime chặn thao tác này và ghi lại: *"Composable này đang đọc phiên bản dữ liệu nào trong Snapshot hiện tại"*.
2. **`writable(this, snapshot)`:** Khi có lệnh gán `state.value = newValue`, Compose không ghi đè trực tiếp lên dữ liệu đang được đọc bởi luồng khác. Nó tạo ra một `StateRecord` phiên bản mới gắn liền với Snapshot của luồng hiện tại.
3. **`apply()`:** Khi Snapshot hoàn thành, nó áp dụng các thay đổi vào `GlobalSnapshot`. Nếu không có xung đột, hệ thống gửi thông báo Invalidate đến toàn bộ cây giao diện!

### 2.2 Các chính sách biến đổi: `SnapshotMutationPolicy`

Khi bạn gọi `mutableStateOf(value, policy)`, Compose cho phép bạn tùy biến cách nhận diện sự thay đổi:

```kotlin
interface SnapshotMutationPolicy<T> {
    fun equivalent(a: T, b: T): Boolean
}
```

Compose cung cấp sẵn 3 chính sách:
1. **`structuralEqualityPolicy()` (Mặc định):** Sử dụng toán tử `==` (so sánh nội dung qua `equals`). Nếu giá trị mới bằng giá trị cũ, không kích hoạt Recomposition.
2. **`referentialEqualityPolicy()`:** Sử dụng toán tử `===` (so sánh địa chỉ con trỏ ô nhớ). Chỉ kích hoạt Recomposition khi tham chiếu đối tượng thay đổi.
3. **`neverEqualPolicy()`:** Luôn trả về `false`. Mọi thao tác gán (`state.value = ...`) đều bị coi là có thay đổi, kích hoạt Recomposition bất kể giá trị mới có giống hệt giá trị cũ hay không.

---

## 3. Bài toán & Kiến trúc (Problem Statement & Architecture)

### 3.1 Bài toán "Bẫy Unstable" của `List<T>`

Hãy quan sát hàm Composable tưởng chừng rất tối ưu sau:

```kotlin
data class Contact(val name: String, val phone: String)

@Composable
fun ContactList(contacts: List<Contact>) {
    LazyColumn {
        items(contacts) { ContactRow(it) }
    }
}
```

Trước Kotlin 2.0, khi Composable cha bị Recompose, hàm `ContactList` **luôn luôn bị Recompose lại**, dù danh sách `contacts` không hề thay đổi một ký tự nào!

**Tại sao Compose Compiler lại đánh giá `List<T>` là UNSTABLE?**
- Trong Kotlin, `List` chỉ là một interface chỉ đọc (read-only), **không phải là bất biến (immutable)**.
- Một đối tượng `val list: List<Contact>` trong thực tế có thể là một `ArrayList` khả biến bị ép kiểu. Một luồng khác có thể âm thầm gọi `(list as ArrayList).add(...)` làm thay đổi nội dung bên trong mà không làm đổi tham chiếu!
- Do không thể đảm bảo chắc chắn nội dung bên trong có bất biến 100% hay không, Compose Compiler buộc phải gắn nhãn an toàn: **`List<T>` là Unstable**, dẫn đến Composable **mất khả năng Skip (Unskippable)**.

### 3.2 Bảng phân loại tính ổn định (Stability Classification)

| Kiểu dữ liệu | Nhãn của Compose Compiler | Khả năng Skip Recomposition |
| :--- | :---: | :---: |
| `Int`, `Float`, `String`, `Boolean`, Enum | **Stable (Primitive)** | ✅ Có thể Skip |
| Function Lambdas: `() -> Unit` | **Stable** | ✅ Có thể Skip |
| Data class chỉ chứa các thuộc tính `val` có kiểu Stable | **Stable** | ✅ Có thể Skip |
| Data class chứa bất kỳ thuộc tính `var` nào | **Unstable** | ❌ Không thể Skip |
| Collection Interfaces: `List<T>`, `Set<T>`, `Map<K, V>` | **Unstable** | ❌ Không thể Skip (Trước Kotlin 2.0) |
| `ImmutableList<T>`, `PersistentList<T>` | **Stable** | ✅ Có thể Skip |

---

## 4. Thực hành tốt nhất: Tối ưu Stability & Strong Skipping Mode

### 4.1 Bước ngoặt lớn: Strong Skipping Mode trong Kotlin 2.0+

Kể từ **Kotlin 2.0** với Compose Compiler mới (được tích hợp trực tiếp vào Kotlin Repo):
- **Strong Skipping Mode** được kích hoạt mặc định!
- **Sự thay đổi luật chơi:** Giờ đây, Compose Compiler cho phép **Skip cả các Composable có tham số Unstable**!
- Thay vì bắt buộc so sánh bằng `equals` (vốn nguy hiểm với Unstable types), Compose sẽ sử dụng phép so sánh tham chiếu **`===` (Instance Equality)** đối với các tham số Unstable. Nếu bạn truyền cùng một instance `List` giữa các lần Recompose, Composable sẽ được Skip thành công!

### 4.2 Các quy tắc vàng để tối ưu hóa Model

Dù có Strong Skipping Mode, để đảm bảo ứng dụng đạt độ mượt mà cao nhất:

> [!TIP]
> 1. **Luôn sử dụng `val` cho tất cả các thuộc tính của UI State:** Tuyệt đối không để thuộc tính `var` trong data class biểu diễn giao diện.
> 2. **Sử dụng `@Immutable` hoặc `@Stable` khi cần thiết:**
>    - `@Immutable`: Cam kết danh dự với Compiler rằng toàn bộ các thuộc tính của class sẽ không bao giờ thay đổi sau khi được khởi tạo.
>    - `@Stable`: Cam kết rằng nếu thuộc tính có biến đổi, nó sẽ phát tín hiệu thông qua Compose State.
> 3. **Tận dụng `kotlinx.collections.immutable`:**
>    - Sử dụng `ImmutableList<T>` hoặc `PersistentList<T>` thay cho `List<T>` thông thường trong các Model quan trọng.

---

## 5. Mã nguồn Thực tế (Implementation)

### 5.1 Sử dụng `@Immutable` và `ImmutableList` để đảm bảo 100% Skippable

```kotlin
package com.example.compose.state.stability

import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.Immutable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

// 1. Gắn nhãn @Immutable để cam kết tính bất biến với Compose Compiler
@Immutable
data class ArticleItem(
    val id: String,
    val title: String,
    val author: String
)

// 2. Composable nhận ImmutableList: Đảm bảo khả năng Skip 100%
@Composable
fun ArticleFeedList(
    articles: ImmutableList<ArticleItem>,
    modifier: Modifier = Modifier
) {
    LazyColumn(modifier = modifier) {
        items(
            items = articles,
            key = { it.id } // Luôn định nghĩa key ổn định cho Lazy Layouts
        ) { article ->
            ArticleRow(article = article)
            HorizontalDivider()
        }
    }
}

@Composable
fun ArticleRow(
    article: ArticleItem,
    modifier: Modifier = Modifier
) {
    Text(
        text = "${article.title} - ${article.author}",
        modifier = modifier.padding(16.dp)
    )
}
```

### 5.2 Tùy biến `SnapshotMutationPolicy` cho tọa độ Game / Đồ họa

Trong trường hợp cần tối ưu hóa không kích hoạt Recomposition khi tọa độ dịch chuyển chưa vượt qua ngưỡng dung sai (Tolerance):

```kotlin
package com.example.compose.state.stability

import androidx.compose.runtime.SnapshotMutationPolicy
import androidx.compose.runtime.mutableStateOf
import kotlin.math.abs

data class Point2D(val x: Float, val y: Float)

/**
 * Chính sách tùy biến: Chỉ coi là có thay đổi khi tọa độ dịch chuyển quá 2 pixel
 */
class ToleranceMutationPolicy(private val tolerance: Float = 2.0f) : SnapshotMutationPolicy<Point2D> {
    override fun equivalent(a: Point2D, b: Point2D): Boolean {
        return abs(a.x - b.x) <= tolerance && abs(a.y - b.y) <= tolerance
    }
}

// Khởi tạo State sử dụng policy tùy biến
val cursorPosition = mutableStateOf(
    value = Point2D(0f, 0f),
    policy = ToleranceMutationPolicy(tolerance = 5.0f)
)
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Tại sao `@Immutable` và `@Stable` được gọi là "Hợp đồng danh dự" (Honor Contract)?
- **Trả lời:** Compose Compiler **hoàn toàn tin tưởng** lập trình viên khi bạn gắn annotation `@Immutable` hoặc `@Stable` lên một class. Nó sẽ không thực hiện bất kỳ kiểm tra mã nguồn tầng sâu nào để xác minh xem bạn có thực sự viết code bất biến hay không.
- Nếu bạn gắn nhãn `@Immutable` lên một class có chứa các thuộc tính `var` hoặc một `ArrayList` bị đột biến âm thầm:
  - Compose sẽ tin rằng đối tượng không bao giờ đổi và **bỏ qua Recomposition (Skip)**!
  - Kết quả: Dữ liệu bên dưới đã thay đổi nhưng giao diện trên màn hình **không bao giờ cập nhật**, gây ra lỗi hiển thị sai lệch cực kỳ khó debug!

### Q2: Làm cách nào để xuất báo cáo Compose Compiler Metrics kiểm tra độ ổn định của toàn bộ dự án?
- **Cách cấu hình trong Gradle (`build.gradle.kts`):**
  ```kotlin
  composeCompiler {
      enableStrongSkippingMode = true
      reportsDestination = layout.buildDirectory.dir("compose_compiler")
      metricsDestination = layout.buildDirectory.dir("compose_compiler")
  }
  ```
- Sau khi chạy lệnh `./gradlew assembleRelease`, Compose Compiler sẽ xuất ra các file văn bản trong thư mục `build/compose_compiler/`:
  - `app_release-classes.txt`: Liệt kê tất cả các data class và nhãn `stable` / `unstable`.
  - `app_release-composables.txt`: Báo cáo chi tiết từng Composable có đạt chuẩn `skippable` và `restartable` hay không.

### Q3: Lambda functions có thể khiến Composable bị mất khả năng Skip không?
- **Trả lời:** Có, nếu lambda vô tình chụp (capture) một biến Unstable từ phạm vi bên ngoài!
- Nếu lambda chụp một biến Unstable, bản thân instance của lambda đó sẽ được tạo mới ở mỗi lần Recomposition, dẫn đến phép so sánh tham chiếu bị thất bại và buộc Composable nhận lambda phải Recompose.
- **Giải pháp:** Sử dụng phương thức tham chiếu (`this::onItemClick`) hoặc đảm bảo tất cả các biến mà lambda chụp đều là Stable.
