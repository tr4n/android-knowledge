# Bài 12 — Recomposition & Stability: @Stable, @Immutable & Compiler Metrics

> **Module:** 3 — Jetpack Compose Foundation  
> **Prerequisite:** [Bài 11 — Compose State: remember, rememberSaveable, State Hoisting](11-compose-state.md)  
> **Official Docs:**
> - [Compose Performance — Stability](https://developer.android.com/develop/ui/compose/performance/stability)
> - [Jetpack Compose Compiler Metrics](https://github.com/androidx/androidx/blob/androidx-main/compose/compiler/design/compiler-metrics.md)
> - [Strong Skipping Mode in Compose](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Recomposition** là quá trình Compose Compiler thực thi lại các hàm `@Composable` khi dữ liệu đầu vào (State hoặc Parameters) bị biến đổi để cập nhật cây giao diện.

Tuy nhiên, nếu mọi Composable đều bị vẽ lại mỗi khi có một thay đổi nhỏ ở tầng cha, ứng dụng sẽ bị giật lag và drop frame nghiêm trọng. Để đạt được hiệu năng 60fps/120fps mượt mà, Compose áp dụng cơ chế **Smart Recomposition (Tái bố cục thông minh)** dựa trên hệ thống **Stability (Độ ổn định của kiểu dữ liệu)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COMPOSE STABILITY SYSTEM                        │
│                                                                        │
│   1. Stable Types (Kiểu dữ liệu ổn định)                               │
│      - Bất biến (Immutable) hoặc thông báo được khi mutate             │
│      - Hai giá trị equals() nhau thì luôn được coi là giống nhau      │
│      - Cho phép Compose BỎ QUA (SKIP) Composable nếu param không đổi   │
│                                                                        │
│   2. Unstable Types (Kiểu dữ liệu không ổn định)                       │
│      - Dữ liệu có thể bị thay đổi ngầm mà Compose không hề biết        │
│      - Compose BUỘC PHẢI CHẠY LẠI (UNSKIPPABLE) Composable mỗi lần     │
│        tầng cha Recompose, dù param trông có vẻ giống nhau!            │
│                                                                        │
│   3. Annotations can thiệp:                                            │
│      - @Immutable: Cam kết 100% không bao giờ thay đổi sau khi tạo     │
│      - @Stable: Cam kết có tính chất ổn định và theo dõi được          │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm Skippable vs Restartable

Khi bạn mở file báo cáo của Compose Compiler (`app_release-composables.txt`), mỗi Composable sẽ được gán nhãn:
- **`restartable`**: Composable này có thể được gọi lại một cách độc lập khi State bên trong nó biến động.
- **`skippable`**: Composable này **có thể được bỏ qua hoàn toàn** nếu tất cả các tham số truyền vào đều là `Stable` và không hề thay đổi giá trị so với lần vẽ trước!

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Compose Compiler Stability Inference (Quy tắc suy luận của Compiler)

Làm thế nào Compose Compiler biết một kiểu dữ liệu là Stable hay Unstable?

```
BẢNG SUY LUẬN TÍNH ỔN ĐỊNH CỦA COMPOSE COMPILER:
┌────────────────────────────────────────────────────────┬──────────────┐
│ Kiểu dữ liệu                                           │ Kết luận     │
├────────────────────────────────────────────────────────┼──────────────┤
│ Nguyên thủy: Int, Float, Double, Boolean, Long...      │ ✅ STABLE     │
│ Chuỗi ký tự: String                                    │ ✅ STABLE     │
│ Functional Types: () -> Unit, (Int) -> String          │ ✅ STABLE     │
│ Compose Snapshot States: State<T>, MutableState<T>     │ ✅ STABLE     │
│ Data class chỉ chứa các trường 'val' kiểu Stable       │ ✅ STABLE     │
├────────────────────────────────────────────────────────┼──────────────┤
│ Class có chứa bất kỳ trường 'var' nào                  │ ❌ UNSTABLE   │
│ Standard Collection Interfaces: List<T>, Set<T>, Map<T>│ ❌ UNSTABLE   │
│ Classes từ external libraries không biên dịch Compose  │ ❌ UNSTABLE   │
└────────────────────────────────────────────────────────┴──────────────┘
```

---

### 2.2 Vấn nạn chí mạng: Tại sao `List<T>` trong Kotlin lại là UNSTABLE?

Đây là câu hỏi khiến hơn 90% kỹ sư Android bất ngờ: **Tại sao `val items: List<String>` là bất biến mà lại bị coi là Unstable?**

```
BẢN CHẤT DƯỚI JVM RUNTIME:

Code Kotlin của bạn:
val items: List<String> = listOf("A", "B")

Bản chất thực tế trên máy ảo JVM:
List<T> chỉ là một INTERFACE (Read-only view),
nhưng đối tượng thực tế nằm trong RAM có thể là java.util.ArrayList (KHẢ BIẾN)!

Kịch bản Race Condition:
Thread 1: Truyền List vào Composable ItemList(items)
Thread 2: (items as ArrayList).add("C") ◄── SỬA ĐỔI NGẦM TRONG BỘ NHỚ!
          Nhưng địa chỉ tham chiếu vùng nhớ (Memory Reference) KHÔNG ĐỔI!

HẬU QUẢ:
Compose Compiler không thể tin tưởng interface List<T>.
Do đó, Compiler đánh dấu List<T> là UNSTABLE!
=> MỌI COMPOSABLE NHẬN List<T> ĐỀU TRỞ THÀNH "NOT SKIPPABLE"!
=> KHI TẦNG CHA RECOMPOSE, DANH SÁCH BỊ VẼ LẠI TOÀN BỘ DÙ KHÔNG CÓ GÌ THAY ĐỔI!
```

#### Giải pháp chuẩn mực:
1. **Dùng thư viện `kotlinx.collections.immutable`:**
   Thay thế `List<T>` bằng `ImmutableList<T>` hoặc `PersistentList<T>`. Compiler nhận diện được đây là kiểu bất biến 100% và đánh dấu `Stable`!
2. **Bọc List trong một `@Immutable data class`:**
   ```kotlin
   @Immutable
   data class ImmutableItemsWrapper(val items: List<Item>)
   ```

---

### 2.3 Sơ đồ kiểm tra Bitmask và cơ chế Skip Composable

```
                CƠ CHẾ QUYẾT ĐỊNH SKIP CỦA COMPOSE COMPILER
                
              Composable nhận tham số: MyCard(title, items, onClick)
                                     │
                                     ▼
                  Tất cả tham số có phải kiểu STABLE không?
                                     │
                        ┌────────────┴────────────┐
                        ▼                         ▼
                     [ YES ]                    [ NO ]
                        │                         │
                        ▼                         ▼
            Các giá trị có giống với       KHÔNG THỂ BỎ QUA!
            lần recompose trước không?     (UNSKIPPABLE)
            (So sánh equals() hoặc CAS)           │
                        │                         │
                 ┌──────┴──────┐                  │
                 ▼             ▼                  │
              [ ĐỔI ]       [ GIỐNG ]             │
                 │             │                  │
                 ▼             ▼                  │
            RECOMPOSE       SKIP TOÀN BỘ! ◄───────┤ (Bắt buộc chạy lại)
            (Vẽ lại node)  (0ms CPU time)         ▼
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi đau Giật Lag khi Cuộn danh sách (Janky Scrolling)

Khi xây dựng danh sách cuộn với `LazyColumn`:
- Nếu mỗi item con nhận một `List` thông thường hoặc một Unstable class, mỗi khi người dùng cuộn ngón tay, Composable cha phát ra frame mới $\rightarrow$ **Tất cả 20 items con đang hiển thị trên màn hình đều bị Recompose lại cùng một lúc**!
- Quá trình này tiêu tốn CPU đột biến, làm tốc độ khung hình tụt từ **120fps xuống còn 35fps**, gây ra hiện tượng giật cục, đứng hình (Jank).

### 3.2 Tối ưu hóa Recomposition mang lại giá trị gì?
1. **Tối ưu pin và nhiệt độ thiết bị:** CPU không phải thực hiện các phép tính diffing và layout vô ích.
2. **Mượt mà tuyệt đối (Fluid 120Hz Animation):** Giúp ứng dụng đạt chuẩn chất lượng cao cấp (Tier-1 Android Apps).

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Khi nào NÊN dùng `@Immutable` và `@Stable`?

```kotlin
// 1. Dùng @Immutable khi TẤT CẢ các trường đều là val và không bao giờ đổi sau khi khởi tạo:
@Immutable
data class UserProfile(
    val id: String,
    val name: String,
    val email: String
)

// 2. Dùng @Stable khi class có thể chứa biến 'var' hoặc MutableState, nhưng cam kết:
// - Khi thuộc tính thay đổi, Compose Snapshot System sẽ được thông báo.
// - Hai instance có cùng dữ liệu thì equals() trả về nhất quán.
@Stable
class ScrollStateTracker {
    var isScrollingUp by mutableStateOf(false)
}
```

---

### 4.2 Cẩn thận với Lambda Instantiations trong Composable

Việc khai báo lambda inline trực tiếp bên trong Composable có thể vô tình tạo ra instance mới ở mỗi lần Recompose, làm Composable con không thể Skip:

```kotlin
// ANTI-PATTERN: Mỗi lần Recompose, một đối tượng Lambda mới được sinh ra trong bộ nhớ!
LazyColumn {
    items(users) { user ->
        // Tạo lambda mới ở mỗi frame -> UserRow bị recompose liên tục!
        UserRow(user = user, onClick = { viewModel.selectUser(user.id) }) 
    }
}

// BEST PRACTICE 1: Dùng Method Reference nếu chữ ký hàm khớp hoàn toàn
UserRow(user = user, onClick = viewModel::onUserSelected)

// BEST PRACTICE 2: Dùng remember bọc Lambda
val onUserClick = remember(user.id) { { viewModel.selectUser(user.id) } }
UserRow(user = user, onClick = onUserClick)
```

---

### 4.3 Cách bật Compose Compiler Metrics trong dự án Gradle

Thêm cấu hình sau vào `build.gradle.kts` (Module: app) để yêu cầu Compose Compiler xuất file báo cáo chi tiết:

```kotlin
// build.gradle.kts
composeCompiler {
    enableMetrics = true
    enableReports = true
}
```

Sau khi chạy lệnh `./gradlew assembleRelease`, mở thư mục `app/build/compose_compiler/` để kiểm tra:
- `app_release-composables.txt`: Danh sách tất cả các Composable kèm nhãn `skippable` / `not skippable`.
- `app_release-classes.txt`: Danh sách các class kèm nhãn `stable` / `unstable`.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Tối ưu hóa danh sách giao dịch tài chính tốc độ cao (**High-Performance Transaction Feed**):
- Loại bỏ triệt để Unstable Types bằng `ImmutableList`.
- Gắn nhãn `@Immutable` cho Model.
- Sử dụng Stable Key trong `LazyColumn`.
- Kiểm chứng tính **Skippable** qua Compose Metrics.

### Bước 1: Khai báo Dependencies

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    // Thư viện tập hợp bất biến chính thức của Kotlin
    implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.7")
}
```

---

### Bước 2: Thiết kế Stable Domain Models & UiState

```kotlin
// Gắn nhãn @Immutable cam kết tính bất biến tuyệt đối
@Immutable
data class Transaction(
    val id: String,
    val title: String,
    val amount: Double,
    val category: String,
    val timestamp: Long
)

// UiState sử dụng ImmutableList thay vì List thông thường!
@Immutable
data class TransactionHistoryUiState(
    val transactions: ImmutableList<Transaction> = persistentListOf(),
    val totalBalance: Double = 0.0,
    val isLoading: Boolean = false
)
```

---

### Bước 3: Triển khai Composable hoàn toàn Skippable

```kotlin
// COMPOSABLE NÀY ĐƯỢC COMPILER ĐÁNH DẤU LÀ: restartable skippable
@Composable
fun TransactionRow(
    transaction: Transaction,
    onClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .clickable { onClick(transaction.id) },
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surface)
    ) {
        Row(
            modifier = Modifier.padding(16.dp).fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column {
                Text(text = transaction.title, style = MaterialTheme.typography.titleMedium)
                Text(
                    text = transaction.category,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.secondary
                )
            }
            Text(
                text = "$${transaction.amount}",
                style = MaterialTheme.typography.titleLarge,
                color = if (transaction.amount >= 0) Color(0xFF2E7D32) else MaterialTheme.colorScheme.error
            )
        }
    }
}
```

---

### Bước 4: Tích hợp vào `LazyColumn` với Stable Keys

```kotlin
@Composable
fun TransactionFeedScreen(
    uiState: TransactionHistoryUiState,
    onTransactionClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Scaffold(
        topBar = { TopAppBar(title = { Text("Lịch sử giao dịch") }) }
    ) { padding ->
        Column(
            modifier = modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {
            // Header hiển thị số dư
            Card(
                colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer),
                modifier = Modifier.fillMaxWidth().padding(bottom = 16.dp)
            ) {
                Column(modifier = Modifier.padding(20.dp)) {
                    Text(text = "Tổng số dư tài khoản", style = MaterialTheme.typography.labelMedium)
                    Text(
                        text = "$${uiState.totalBalance}",
                        style = MaterialTheme.typography.headlineMedium,
                        fontWeight = FontWeight.Bold
                    )
                }
            }

            // Danh sách cuộn cực kỳ mượt mà:
            LazyColumn(
                verticalArrangement = Arrangement.spacedBy(8.dp),
                modifier = Modifier.fillMaxSize()
            ) {
                // QUAN TRỌNG: Cung cấp stable key duy nhất cho từng item
                items(
                    items = uiState.transactions,
                    key = { transaction -> transaction.id }
                ) { transaction ->
                    TransactionRow(
                        transaction = transaction,
                        onClick = onTransactionClick
                    )
                }
            }
        }
    }
}
```

---

### Bước 5: Đọc và giải thích báo cáo Compose Compiler Metrics

Sau khi biên dịch, mở file `app_release-composables.txt`:

```text
// KẾT QUẢ PHÂN TÍCH CỦA TRÌNH BIÊN DỊCH:
restartable skippable fun TransactionRow(
  stable transaction: Transaction
  stable onClick: Function1<String, Unit>
  stable modifier: Modifier? = @static Companion
)

restartable skippable fun TransactionFeedScreen(
  stable uiState: TransactionHistoryUiState
  stable onTransactionClick: Function1<String, Unit>
  stable modifier: Modifier? = @static Companion
)
```
> **Đánh giá của Senior Architect:** Cả 2 hàm Composable đều đạt trạng thái `skippable` tuyệt đối! Khi cuộn màn hình hoặc cập nhật số dư ở Header, các item `TransactionRow` có dữ liệu không đổi sẽ được **BỎ QUA 100%**, không tốn dù chỉ 1 chu kỳ xử lý của CPU.

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Sự khác biệt thực sự giữa `@Stable` và `@Immutable` là gì?
**Trả lời chuẩn bản chất:**
- **`@Immutable`**: Là một cam kết nghiêm ngặt nhất với trình biên dịch rằng: **Đối tượng này và tất cả các trường công khai của nó sẽ KHÔNG BAO GIỜ thay đổi giá trị sau khi được khởi tạo**.
- **`@Stable`**: Là một cam kết ít nghiêm ngặt hơn rằng: Đối tượng này có thể có thuộc tính biến đổi, nhưng **bất kỳ sự biến đổi nào cũng sẽ được thông báo rõ ràng cho Compose Runtime** (ví dụ thông qua `MutableState`). Ngoài ra, hai instance được coi là bằng nhau nếu hàm `equals()` trả về kết quả nhất quán.

#### Q2: Strong Skipping Mode (Chế độ bỏ qua mạnh) trong Kotlin 2.0 hoạt động ra sao?
**Trả lời chuẩn bản chất:**
Trong các phiên bản Compose cũ, nếu Composable nhận một tham số `Unstable`, hàm đó vĩnh viễn không thể Skip. 
Từ Compose Compiler 1.5.4+ và mặc định trong Kotlin 2.0+, **Strong Skipping Mode** thay đổi luật chơi:
- Các Composable có tham số Unstable **vẫn có thể được Skip** nếu tham số đó có cùng địa chỉ tham chiếu vùng nhớ (**Instance Equality `===`**).
- Các lambda không capture biến cũng tự động được bọc trong `remember`.
Tuy nhiên, Strong Skipping Mode **không thể cứu vãn** các trường hợp object được tạo mới ở mỗi frame (Structural Equality `equals()`). Vì vậy, việc thiết kế kiểu dữ liệu Stable vẫn là tiêu chuẩn vàng bắt buộc của một Architect.

#### Q3: Tại sao Composable của tôi có chữ `restartable` nhưng lại ghi `not skippable`?
**Trả lời chuẩn bản chất:**
Vì ít nhất một trong các tham số truyền vào hàm Composable bị suy luận là **Unstable**. Hãy mở file `app_release-classes.txt`, tìm tên class của tham số đó. Bạn sẽ thấy compiler chỉ rõ trường nào đang bị Unstable (thường là do dùng `var`, hoặc dùng interface `List`/`Map`, hoặc import class từ thư viện ngoài chưa biên dịch với Compose Plugin).

---

### 6.2 Lỗi Runtime/Performance thường gặp & Hướng xử lý

| Hiện tượng | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Cuộn LazyColumn bị drop frame (Jank)** | Dùng `List<T>` trong Composable con hoặc không cung cấp `key = { it.id }`. | Chuyển sang `ImmutableList<T>` và luôn khai báo `key` cho `items()`. |
| **Hàm Composable Recompose liên tục không lý do** | Khởi tạo đối tượng mới (ví dụ: `Modifier.padding()`, `Date()`) trực tiếp trong tham số gọi hàm con. | Đưa việc khởi tạo ra ngoài hoặc bọc trong `remember`. |
| **Strong Skipping không hoạt động như mong đợi** | Object truyền vào bị mutate các thuộc tính bên trong mà không tạo instance mới. | Luôn sử dụng immutable data class và hàm `.copy()` để sinh instance mới. |

---

*Bài trước: [11 — Compose State: remember, rememberSaveable, State Hoisting](11-compose-state.md)*  
*Bài tiếp theo: [13 — Side Effects trong Compose: LaunchedEffect, DisposableEffect](13-side-effects.md)*
