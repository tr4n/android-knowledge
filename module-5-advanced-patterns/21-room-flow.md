# Bài 21 — Room + Flow: Reactive Database, InvalidationTracker & Safe Migrations

> **Module:** 5 — Advanced Patterns & Integration  
> **Mức độ:** Senior / Staff Android Architect  
> **Prerequisites:** Module 1 (Coroutines & Flow), Module 2 (Repository Pattern & SSOT), Module 5 (Bài 19 - Hilt DI)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Room Persistence Library** là tầng trừu tượng hóa (Abstraction Layer) chính thức của Google bên trên cơ sở dữ liệu **SQLite** nhúng trong hệ điều hành Android. Room kết hợp sức mạnh kiểm tra cú pháp câu lệnh SQL ngay lúc biên dịch (**Compile-Time SQL Verification**) với khả năng phản ứng thời gian thực (**Reactive Data Stream**) của **Kotlin Flow**.

```
                ┌──────────────────────────────────────────────┐
                │          Jetpack Compose UI Layer            │
                │        collectAsStateWithLifecycle()         │
                └──────────────────────▲───────────────────────┘
                                       │
                              Emits New List<Data>
                                       │
                ┌──────────────────────┴──────────────────────┐
                │             Repository / DAO                │
                │         fun getOrders(): Flow<List<Order>>  │
                └──────────────────────▲───────────────────────┘
                                       │
                      Invalidation Event (Re-query)
                                       │
   ┌───────────────────────────────────┴──────────────────────────────────┐
   │                     ROOM INVALIDATION TRACKER                        │
   │  Theo dõi SQLite Triggers: room_table_modification_trigger           │
   └───────────────────────────────────▲──────────────────────────────────┘
                                       │
                               Writes to WAL file
                                       │
   ┌───────────────────────────────────┴──────────────────────────────────┐
   │                   SQLITE ENGINE (WAL MODE)                           │
   │      app_database.db       app_database.db-wal    (Concurrent R/W)   │
   └──────────────────────────────────────────────────────────────────────┘
```

### Các thuật ngữ cốt lõi:
- **`@Entity`:** Đại diện cho một bảng (Table) trong SQLite. Mỗi trường dữ liệu trong data class tương ứng với một cột (Column).
- **`@Dao` (Data Access Object):** Giao diện định nghĩa các thao tác truy vấn và biến đổi dữ liệu (CRUD). DAO là ranh giới duy nhất tương tác trực tiếp với cơ sở dữ liệu.
- **`@Database`:** Lớp cơ sở trừu tượng kế thừa `RoomDatabase`, đóng vai trò là điểm kết nối chính và quản lý phiên bản (Schema Version) của cơ sở dữ liệu.
- **Reactive Query (`Flow<T>`):** Truy vấn trả về một Kotlin Flow. Khi bảng tương ứng có bất kỳ thay đổi nào (Insert/Update/Delete), Room sẽ tự động thực thi lại câu truy vấn ngầm và phát (emit) ra danh sách dữ liệu mới nhất.
- **`@Transaction`:** Đảm bảo một nhóm các câu lệnh SQL được thực thi theo nguyên tắc **Nguyên tử (Atomic - ACID)**: Hoặc tất cả đều thành công, hoặc nếu một lệnh lỗi thì toàn bộ thay đổi sẽ được hoàn tác (Rollback). Bắt buộc phải có khi truy vấn các quan hệ nhiều bảng (`@Relation`).
- **Write-Ahead Logging (WAL Mode):** Cơ chế ghi nhật ký trước của SQLite giúp tách biệt luồng Đọc và Ghi, cho phép nhiều luồng đọc đồng thời với một luồng ghi mà không bị khóa (Lock) cơ sở dữ liệu.
- **Schema Hash (`identity_hash`):** Mã băm SHA-256 được Room tạo ra dựa trên cấu trúc các Entity và lưu trữ trong bảng `room_master_table`. Room dùng mã này để xác minh tính toàn vẹn của cơ sở dữ liệu khi mở kết nối.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế phản ứng của Room: `InvalidationTracker`
Làm thế nào Room biết được khi nào dữ liệu trong bảng thay đổi để kích hoạt `Flow` phát ra giá trị mới?
Room **không** liên tục thăm dò (polling) bảng SQLite theo chu kỳ vì điều đó sẽ gây cạn kiệt pin và nghẽn CPU. Thay vào đó, Room triển khai một cơ chế quan sát cấp thấp cực kỳ tinh vi:

1. **Tạo bảng tạm và Triggers nội bộ:**
   - Khi bạn đăng ký một câu truy vấn `Flow<List<Order>>` quan sát bảng `orders`, Room tự động tạo một bảng theo dõi: `room_table_modification_tracker`.
   - Room cài đặt các Trigger trong SQLite (`INSERT`, `UPDATE`, `DELETE`) trên bảng `orders`. Mỗi khi có bản ghi bị thay đổi, Trigger sẽ tự động cập nhật cờ (flag) tương ứng trong bảng theo dõi.
2. **Kích hoạt luồng quan sát ngầm:**
   - Sau bất kỳ thao tác ghi dữ liệu nào (thông qua DAO `@Insert`, `@Update`, `@Upsert`), Room thông báo cho `InvalidationTracker`.
   - `InvalidationTracker` kiểm tra cờ thay đổi trên thread pool của Room. Nếu phát hiện bảng `orders` đã bị biến đổi (invalidated), nó sẽ kích hoạt lại hàm truy vấn SQL ban đầu.
3. **Phát dữ liệu qua Flow:**
   - Dữ liệu mới được đọc từ `Cursor`, chuyển đổi thành Kotlin Data Classes, và phát qua `FlowCollector`.

```
[ INSERT / UPDATE / DELETE ]
            │
            ▼
[ SQLite Trigger: room_table_modification_trigger ]
            │
            ▼ Cập nhật flag
[ room_table_modification_tracker ]
            │
            ▼ Đánh thức
[ Room InvalidationTracker ]
            │
            ▼ Chạy lại câu SELECT trên I/O Thread
[ CursorWindow ──► List<Entity> ]
            │
            ▼
[ Flow<List<Entity>> emit(newList) ] ──► [ ViewModel / UI Recomposition ]
```

### 2.2 SQLite Write-Ahead Logging (WAL) Mode
Mặc định từ Android 9.0 (API 28), Room tự động bật chế độ **WAL (Write-Ahead Logging)**:
- **Chế độ Rollback Journal cũ:** Khi một tiến trình đang ghi dữ liệu, nó sẽ khóa độc quyền toàn bộ tệp database (`EXCLUSIVE lock`). Mọi luồng đọc khác (như UI thread đang load danh sách) đều bị chặn (blocked), gây ra hiện tượng giật khung hình (Jank / ANR).
- **Chế độ WAL hiện đại:** Các thao tác ghi không sửa trực tiếp vào file gốc `database.db`. Thay vào đó, dữ liệu mới được ghi vào một file riêng biệt gọi là `database.db-wal`.
  - Các luồng đọc có thể đọc đồng thời từ file gốc và file WAL mà **hoàn toàn không bị chặn bởi luồng ghi**.
  - Định kỳ, SQLite sẽ đồng bộ dữ liệu từ WAL về database chính (tiến trình **Checkpointing**).

### 2.3 Cơ chế Transaction với `@Relation` và `@Embedded`
Trong SQLite, không có khái niệm "Object lồng Object". Khi bạn định nghĩa một class chứa quan hệ 1-N:
```kotlin
data class OrderWithItems(
    @Embedded val order: OrderEntity,
    @Relation(parentColumn = "orderId", entityColumn = "parentOrderId")
    val items: List<OrderItemEntity>
)
```
Để trả về đối tượng này, bên dưới bytecode Room phải thực thi **ít nhất 2 câu lệnh SQL riêng biệt**:
1. `SELECT * FROM orders WHERE ...`
2. `SELECT * FROM order_items WHERE parentOrderId IN (...)`

Nếu **không có `@Transaction`**, một luồng khác có thể chèn hoặc xóa các item ngay giữa thời điểm câu lệnh 1 và câu lệnh 2 hoàn thành. Hậu quả là kết quả trả về sẽ bị xé mảnh dữ liệu (**Inconsistent / Dirty Read**). Thêm `@Transaction` đảm bảo SQLite giữ một Snapshot nhất quán của toàn bộ cơ sở dữ liệu trong suốt quá trình đọc cả 2 bảng.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi kinh hoàng của Raw SQLite truyền thống
Trước khi có Room, lập trình viên Android phải làm việc trực tiếp với `SQLiteOpenHelper`:
```kotlin
// ❌ ANTI-PATTERN: Raw SQLite với Cursor thủ công
val cursor = db.rawQuery("SELECT id, user_nam, email FROM users", null) // Viết sai chính tả 'user_nam'!
while (cursor.moveToNext()) {
    val id = cursor.getLong(cursor.getColumnIndexOrThrow("id"))
    val name = cursor.getString(1) // Magic number index!
}
cursor.close() // Quên close là rò rỉ CursorWindow!
```
- **Không có kiểm tra lúc biên dịch (Zero Compile-Time Verification):** Sai một dấu phẩy hoặc sai tên cột trong chuỗi String SQL, ứng dụng vẫn build thành công nhưng sẽ lập tức văng lỗi `SQLiteException` khi người dùng mở màn hình!
- **Mapping thủ công dễ lỗi (Cursor Boilerplate):** Phải tự lấy từng index, ép kiểu từng cột; rất dễ sai lệch thứ tự cột khi schema thay đổi.
- **Hoàn toàn không có tính phản ứng (Not Reactive):** Muốn UI tự cập nhật khi dữ liệu thay đổi, lập trình viên phải tự viết hệ thống Observer hoặc `ContentObserver` thủ công cồng kềnh và dễ gây rò rỉ bộ nhớ (Memory Leak).

### 3.2 Giá trị vượt trội của Room + Flow
- **Kiểm tra cú pháp 100% lúc Compile:** Nếu bạn viết sai tên bảng, tên cột hoặc kiểu dữ liệu trong `@Query`, IDE và KSP sẽ báo lỗi đỏ ngay lập tức trước khi build file APK.
- **Tự động chuyển hóa dữ liệu (Type-Safe ORM):** Room tự động sinh mã đọc `Cursor` và map vào Kotlin data class với hiệu năng cực cao.
- **Luồng dữ liệu phản ứng thời gian thực (SSOT Reactive Stream):** Kết hợp trực tiếp với Kotlin Flow. UI chỉ cần `collectAsStateWithLifecycle()` một lần duy nhất; mọi thay đổi từ Network đồng bộ xuống Database sẽ tự động làm mới giao diện người dùng.

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Do's and Don'ts từ Google & Database Architects

#### DO:
1. **Sử dụng `@Upsert` thay cho `@Insert(onConflict = OnConflictStrategy.REPLACE)`:**
   - Kể từ Room 2.5.0, Google giới thiệu `@Upsert`.
   - `@Insert(onConflict = REPLACE)` bên dưới SQLite thực chất là một lệnh `DELETE` sau đó `INSERT`. Việc này sẽ làm thay đổi `rowid`, kích hoạt xóa dây chuyền khóa ngoại (Cascade Delete) không mong muốn.
   - `@Upsert` thực thi `INSERT ... ON CONFLICT DO UPDATE`, bảo toàn toàn vẹn dữ liệu và khóa ngoại.
2. **Luôn đánh chỉ mục (`@Index`) cho các cột dùng trong `WHERE`, `JOIN` và `ORDER BY`:**
   - Mặc định chỉ có Khóa chính (`@PrimaryKey`) được đánh chỉ mục.
   - Nếu bạn thường xuyên truy vấn `WHERE userId = :userId` hoặc `WHERE status = 'PENDING'`, bắt buộc phải khai báo `indices = [Index("userId")]` để tránh việc SQLite phải quét toàn bộ bảng (Full Table Scan O(N)).
3. **Luôn viết Automated Migration Tests:** Đảm bảo dữ liệu người dùng cũ không bao giờ bị mất hoặc bị crash khi người dùng nâng cấp ứng dụng từ phiên bản cũ lên phiên bản mới.
4. **Sử dụng `.distinctUntilChanged()` khi cần thiết:** Mặc dù Room rất thông minh, nhưng nếu bảng có cập nhật ở một hàng khác không liên quan đến kết quả của câu query, Room vẫn có thể re-emit. Đặt `.distinctUntilChanged()` trong Repository giúp ngăn chặn Recomposition dư thừa trên giao diện.

#### DON'T:
1. **TUYỆT ĐỐI KHÔNG dùng `fallbackToDestructiveMigration()` trên Production:**
   - Phương thức này sẽ **XÓA SẠCH 100% DATABASE CỦA NGƯỜI DÙNG** khi phát hiện version database tăng lên mà không có file Migration tương ứng! Chỉ được phép dùng hàm này trong giai đoạn phát triển ban đầu (Local Dev).
2. **KHÔNG thực thi DAO Blocking queries trên Main Thread:**
   - Room chặn đứng việc gọi query đồng bộ trên Main Thread bằng `IllegalStateException: Cannot access database on the main thread`. Luôn dùng `suspend fun` hoặc trả về `Flow<T>`.
3. **KHÔNG lạm dụng `@Relation` cho các cấu trúc dữ liệu quá sâu:** Nếu một đơn hàng có danh sách sản phẩm, mỗi sản phẩm có danh sách đánh giá, mỗi đánh giá có danh sách bình luận... việc lồng `@Relation` 4-5 tầng sẽ làm suy giảm nghiêm trọng hiệu năng truy vấn của SQLite.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Chúng ta sẽ xây dựng hệ thống cơ sở dữ liệu quan hệ phức tạp: **Quản lý Đơn hàng & Chi tiết Sản phẩm (E-Commerce Order System)**, xử lý quan hệ 1-N, giao dịch `@Transaction`, và quy trình Migration từ Version 1 lên Version 2 an toàn.

### Bước 1: Khai báo Entities & Quan hệ 1-N

Tạo tệp `data/local/OrderEntities.kt`:

```kotlin
package com.example.roomflow.data.local

import androidx.room.*
import java.util.Date

// 1. Bảng Đơn hàng (Orders Table)
@Entity(
    tableName = "orders",
    indices = [Index(value = ["customerId"]), Index(value = ["orderDate"])]
)
data class OrderEntity(
    @PrimaryKey
    val orderId: String,
    val customerId: String,
    val orderDate: Long, // Lưu timestamp miligiây
    val totalAmount: Double,
    val status: String // "PENDING", "PAID", "SHIPPED", "CANCELLED"
)

// 2. Bảng Chi tiết mặt hàng (Order Items Table)
@Entity(
    tableName = "order_items",
    foreignKeys = [
        ForeignKey(
            entity = OrderEntity::class,
            parentColumns = ["orderId"],
            childColumns = ["parentOrderId"],
            onDelete = ForeignKey.CASCADE // Xóa đơn hàng thì tự động xóa sạch các item con
        )
    ],
    indices = [Index(value = ["parentOrderId"])]
)
data class OrderItemEntity(
    @PrimaryKey(autoGenerate = true)
    val itemId: Long = 0,
    val parentOrderId: String,
    val productName: String,
    val quantity: Int,
    val price: Double
)

// 3. Quan hệ 1-N giữa Order và OrderItems
data class OrderWithItems(
    @Embedded
    val order: OrderEntity,

    @Relation(
        parentColumn = "orderId",
        entityColumn = "parentOrderId"
    )
    val items: List<OrderItemEntity>
)
```

### Bước 2: Triển khai DAO với Reactive Flow, `@Upsert` và `@Transaction`

Tạo tệp `data/local/OrderDao.kt`:

```kotlin
package com.example.roomflow.data.local

import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface OrderDao {

    // 1. Reactive Query quan sát danh sách đơn hàng theo trạng thái
    @Transaction
    @Query("SELECT * FROM orders WHERE status = :status ORDER BY orderDate DESC")
    fun getOrdersWithItemsByStatus(status: String): Flow<List<OrderWithItems>>

    // 2. Reactive Query lấy chi tiết một đơn hàng duy nhất
    @Transaction
    @Query("SELECT * FROM orders WHERE orderId = :orderId")
    fun getOrderWithItemsById(orderId: String): Flow<OrderWithItems?>

    // 3. Atomic Upsert: Chèn hoặc cập nhật Order mà không thay đổi rowid
    @Upsert
    suspend fun upsertOrder(order: OrderEntity)

    @Upsert
    suspend fun upsertOrderItems(items: List<OrderItemEntity>)

    // 4. Atomic Transaction: Lưu toàn bộ Đơn hàng và Danh sách Item trong 1 Transaction duy nhất
    @Transaction
    suspend fun insertCompleteOrder(order: OrderEntity, items: List<OrderItemEntity>) {
        upsertOrder(order)
        upsertOrderItems(items)
    }

    // 5. Cập nhật trạng thái đơn hàng
    @Query("UPDATE orders SET status = :newStatus WHERE orderId = :orderId")
    suspend fun updateOrderStatus(orderId: String, newStatus: String)

    // 6. Xóa đơn hàng (Tự động kích hoạt CASCADE xóa order_items)
    @Query("DELETE FROM orders WHERE orderId = :orderId")
    suspend fun deleteOrder(orderId: String)
}
```

### Bước 3: Cấu hình Room Database & Kịch bản Migration an toàn (v1 $\to$ v2)

Giả sử ở phiên bản 1 (v1), bảng `orders` chỉ có 5 cột. Khi nâng cấp lên phiên bản 2 (v2), doanh nghiệp yêu cầu thêm 1 cột mới: `discountCode` (chuỗi nullable).

Tạo tệp `data/local/AppDatabase.kt`:

```kotlin
package com.example.roomflow.data.local

import androidx.room.Database
import androidx.room.RoomDatabase
import androidx.room.migration.Migration
import androidx.sqlite.db.SupportSQLiteDatabase

@Database(
    entities = [OrderEntity::class, OrderItemEntity::class],
    version = 2,
    exportSchema = true // Xuất file json schema để phục vụ Automated Migration Testing
)
abstract class AppDatabase : RoomDatabase() {

    abstract fun orderDao(): OrderDao

    companion object {
        const val DATABASE_NAME = "enterprise_ecommerce.db"

        // CHIẾN LƯỢC MIGRATION THỦ CÔNG AN TOÀN TỪ V1 LÊN V2
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                // Thêm cột mới discountCode vào bảng orders mà không làm mất dữ liệu hiện có
                db.execSQL("ALTER TABLE orders ADD COLUMN discountCode TEXT DEFAULT NULL")
            }
        }
    }
}
```

Cung cấp Room Database thông qua Hilt Module trong `di/DatabaseModule.kt`:

```kotlin
package com.example.roomflow.di

import android.content.Context
import androidx.room.Room
import com.example.roomflow.data.local.AppDatabase
import com.example.roomflow.data.local.OrderDao
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideAppDatabase(
        @ApplicationContext context: Context
    ): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            AppDatabase.DATABASE_NAME
        )
            .addMigrations(AppDatabase.MIGRATION_1_2)
            // TUYỆT ĐỐI KHÔNG GỌI: .fallbackToDestructiveMigration() ở môi trường Production!
            .build()
    }

    @Provides
    @Singleton
    fun provideOrderDao(database: AppDatabase): OrderDao {
        return database.orderDao()
    }
}
```

### Bước 4: Tầng Repository & ViewModel tích hợp Flow

Tạo tệp `ui/OrderViewModel.kt`:

```kotlin
package com.example.roomflow.ui

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.roomflow.data.local.OrderDao
import com.example.roomflow.data.local.OrderEntity
import com.example.roomflow.data.local.OrderItemEntity
import com.example.roomflow.data.local.OrderWithItems
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import java.util.UUID
import javax.inject.Inject

sealed interface OrderUiState {
    data object Loading : OrderUiState
    data class Success(val orders: List<OrderWithItems>) : OrderUiState
    data class Empty(val message: String) : OrderUiState
}

@HiltViewModel
class OrderViewModel @Inject constructor(
    private val orderDao: OrderDao
) : ViewModel() {

    // Bộ lọc trạng thái đơn hàng phản ứng
    private val _selectedStatus = MutableStateFlow("PENDING")
    val selectedStatus: StateFlow<String> = _selectedStatus.asStateFlow()

    // Chuyển hóa Room Flow thành UI State
    val uiState: StateFlow<OrderUiState> = _selectedStatus
        .flatMapLatest { status ->
            orderDao.getOrdersWithItemsByStatus(status)
                .distinctUntilChanged() // Ngăn ngừa phát lại nếu nội dung không đổi
        }
        .map { orderList ->
            if (orderList.isEmpty()) {
                OrderUiState.Empty("Không có đơn hàng nào ở trạng thái ${_selectedStatus.value}")
            } else {
                OrderUiState.Success(orderList)
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = OrderUiState.Loading
        )

    fun changeStatusFilter(newStatus: String) {
        _selectedStatus.value = newStatus
    }

    fun createSampleOrder() {
        viewModelScope.launch {
            val orderId = UUID.randomUUID().toString().take(8)
            val order = OrderEntity(
                orderId = orderId,
                customerId = "CUST_001",
                orderDate = System.currentTimeMillis(),
                totalAmount = 550.0,
                status = "PENDING"
            )
            val items = listOf(
                OrderItemEntity(parentOrderId = orderId, productName = "Màn hình 4K LG", quantity = 1, price = 450.0),
                OrderItemEntity(parentOrderId = orderId, productName = "Cáp HDMI Ultra", quantity = 2, price = 50.0)
            )
            // Giao dịch nguyên tử: Chèn cả Order và Items
            orderDao.insertCompleteOrder(order, items)
        }
    }

    fun markAsPaid(orderId: String) {
        viewModelScope.launch {
            orderDao.updateOrderStatus(orderId, "PAID")
        }
    }

    fun deleteOrder(orderId: String) {
        viewModelScope.launch {
            orderDao.deleteOrder(orderId)
        }
    }
}
```

### Bước 5: Jetpack Compose UI Reactive Render

Tạo tệp `ui/OrderScreen.kt`:

```kotlin
package com.example.roomflow.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.example.roomflow.data.local.OrderWithItems

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OrderScreen(
    viewModel: OrderViewModel = hiltViewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    val currentFilter by viewModel.selectedStatus.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Room + Flow: Quản lý Đơn Hàng") }
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = viewModel::createSampleOrder) {
                Icon(Icons.Default.Add, contentDescription = "Tạo đơn hàng mới")
            }
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
        ) {
            // Bộ lọc Status tabs
            FilterStatusTabs(
                selected = currentFilter,
                onSelect = viewModel::changeStatusFilter
            )

            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(16.dp),
                contentAlignment = Alignment.Center
            ) {
                when (val current = state) {
                    is OrderUiState.Loading -> CircularProgressIndicator()
                    is OrderUiState.Empty -> Text(current.message, style = MaterialTheme.typography.bodyLarge)
                    is OrderUiState.Success -> {
                        LazyColumn(
                            modifier = Modifier.fillMaxSize(),
                            verticalArrangement = Arrangement.spacedBy(12.dp)
                        ) {
                            items(current.orders, key = { it.order.orderId }) { orderWithItems ->
                                OrderCard(
                                    orderWithItems = orderWithItems,
                                    onMarkPaid = { viewModel.markAsPaid(orderWithItems.order.orderId) },
                                    onDelete = { viewModel.deleteOrder(orderWithItems.order.orderId) }
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun FilterStatusTabs(
    selected: String,
    onSelect: (String) -> Unit
) {
    val statuses = listOf("PENDING", "PAID", "SHIPPED")
    PrimaryTabRow(selectedTabIndex = statuses.indexOf(selected).coerceAtLeast(0)) {
        statuses.forEach { status ->
            Tab(
                selected = selected == status,
                onClick = { onSelect(status) },
                text = { Text(status) }
            )
        }
    }
}

@Composable
fun OrderCard(
    orderWithItems: OrderWithItems,
    onMarkPaid: () -> Unit,
    onDelete: () -> Unit
) {
    val order = orderWithItems.order
    val items = orderWithItems.items

    Card(modifier = Modifier.fillMaxWidth()) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text("Đơn hàng #${order.orderId}", style = MaterialTheme.typography.titleMedium)
                Text(
                    "$${order.totalAmount}",
                    style = MaterialTheme.typography.titleLarge,
                    color = MaterialTheme.colorScheme.primary
                )
            }

            Spacer(modifier = Modifier.height(8.dp))
            HorizontalDivider()
            Spacer(modifier = Modifier.height(8.dp))

            Text("Sản phẩm (${items.size}):", style = MaterialTheme.typography.labelMedium)
            items.forEach { item ->
                Text(
                    "- ${item.productName} (x${item.quantity}): $${item.price}",
                    style = MaterialTheme.typography.bodySmall
                )
            }

            Spacer(modifier = Modifier.height(12.dp))
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.End,
                verticalAlignment = Alignment.CenterVertically
            ) {
                OutlinedButton(onClick = onDelete) {
                    Text("Xóa", color = MaterialTheme.colorScheme.error)
                }
                if (order.status == "PENDING") {
                    Spacer(modifier = Modifier.width(8.dp))
                    Button(onClick = onMarkPaid) {
                        Text("Đã thanh toán")
                    }
                }
            }
        }
    }
}
```

### Bước 6: Automated Database & Migration Testing với `MigrationTestHelper`

Tạo tệp test `MigrationTest.kt` trong thư mục `androidTest`:

```kotlin
package com.example.roomflow

import androidx.room.testing.MigrationTestHelper
import androidx.sqlite.db.framework.FrameworkSQLiteOpenHelperFactory
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.platform.app.InstrumentationRegistry
import com.example.roomflow.data.local.AppDatabase
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class MigrationTest {

    private val TEST_DB = "migration-test.db"

    @get:Rule
    val helper: MigrationTestHelper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java,
        emptyList(),
        FrameworkSQLiteOpenHelperFactory()
    )

    @Test
    fun migrate1To2_containsCorrectDataAndNewColumn() {
        // 1. Tạo database ở Version 1 và chèn dữ liệu mẫu
        var db = helper.createDatabase(TEST_DB, 1).apply {
            execSQL(
                """
                INSERT INTO orders (orderId, customerId, orderDate, totalAmount, status)
                VALUES ('ORD_01', 'CUST_A', 1700000000, 250.0, 'PENDING')
                """.trimIndent()
            )
            close()
        }

        // 2. Chạy Migration lên Version 2 và tự động kiểm tra Schema
        db = helper.runMigrationsAndValidate(TEST_DB, 2, true, AppDatabase.MIGRATION_1_2)

        // 3. Xác minh dữ liệu cũ không bị mất và cột mới đã được tạo với giá trị mặc định
        val cursor = db.query("SELECT orderId, totalAmount, discountCode FROM orders WHERE orderId = 'ORD_01'")
        assertTrue(cursor.moveToFirst())

        val id = cursor.getString(0)
        val total = cursor.getDouble(1)
        val discount = cursor.getString(2)

        assertEquals("ORD_01", id)
        assertEquals(250.0, total, 0.01)
        assertEquals(null, discount) // Cột mới discountCode mặc định là NULL

        cursor.close()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior / Staff Android Architect

#### Câu hỏi 1: Tại sao Room Flow đôi khi lại phát ra (emit) danh sách mới ngay cả khi hàng (row) bị cập nhật không thuộc kết quả của câu lệnh WHERE?
**Trả lời:**
- Đây là cơ chế thiết kế của `InvalidationTracker`. Room theo dõi thay đổi ở **Cấp độ Bảng (Table-Level Invalidation)** chứ không theo dõi từng dòng riêng lẻ (Row-Level Tracking) nhằm tránh việc tiêu tốn bộ nhớ RAM để lập chỉ mục cho từng rowid.
- Khi có bất kỳ hành động ghi nào xảy ra trên bảng `orders`, InvalidationTracker phát hiện bảng đã thay đổi và thông báo cho tất cả các câu truy vấn Flow đang quan sát bảng `orders` thực thi lại câu lệnh `SELECT`.
- **Giải pháp tối ưu:** Trong Repository, luôn sử dụng toán tử `.distinctUntilChanged()` trên Flow. Nếu câu lệnh `SELECT` thực thi lại nhưng trả về danh sách đối tượng có nội dung giống hệt danh sách cũ (`equals == true`), Flow sẽ lập tức chặn lại và không phát dữ liệu thừa lên UI, ngăn ngừa 100% việc Recomposition dư thừa.

#### Câu hỏi 2: Sự khác nhau giữa AutoMigration và Manual Migration trong Room là gì? Khi nào bắt buộc phải viết Manual Migration?
**Trả lời:**
- **AutoMigration (từ Room 2.4+):** Áp dụng annotation `@Database(autoMigrations = [AutoMigration(from = 1, to = 2)])`. Room tự động so sánh hai file JSON schema xuất ra và sinh câu lệnh SQL cho các thay đổi đơn giản: Thêm cột mới nullable hoặc có default value, thêm bảng mới.
- **Manual Migration (Bắt buộc):** Khi cấu trúc thay đổi phức tạp mà SQLite không thể tự động suy luận:
  1. Đổi tên cột hoặc đổi kiểu dữ liệu của cột cũ (ví dụ: đổi từ `INTEGER` sang `TEXT`).
  2. Gộp 2 bảng thành 1 bảng hoặc tách 1 bảng thành 2 bảng quan hệ.
  3. Cần chuyển đổi (Transform/Migrate) dữ liệu cũ sang định dạng dữ liệu mới trong lúc nâng cấp.

#### Câu hỏi 3: Tại sao `@Transaction` lại bắt buộc khi một hàm DAO trả về một đối tượng chứa `@Relation`?
**Trả lời:**
- Khi bạn truy vấn một đối tượng chứa `@Relation`, Room không thể dùng 1 câu lệnh JOIN duy nhất vì cấu trúc phân cấp đối tượng không tương thích với bảng 2 chiều phẳng của SQLite. Room buộc phải thực thi 2 hoặc nhiều câu lệnh `SELECT` nối tiếp nhau.
- Nếu không có `@Transaction`, SQLite sẽ thực thi các câu lệnh này trong các Snapshot khác nhau. Nếu một luồng ghi khác thay đổi bảng con ngay giữa 2 lần `SELECT`, dữ liệu của đối tượng trả về sẽ bị sai lệch hoàn toàn (Data Inconsistency / Phantom Reads).

### 6.2 Bảng gỡ rối các lỗi thực tế (Troubleshooting Matrix)

| Vấn đề / Lỗi thực tế | Nguyên nhân gốc rễ (Root Cause) | Giải pháp triệt để (Solution) |
|---|---|---|
| Crash: `IllegalStateException: Room cannot verify the data integrity` | Bạn vừa sửa một `@Entity` (thêm/sửa cột) nhưng quên tăng `version` của Database hoặc quên thêm file Migration | Tăng biến `version` trong `@Database` và viết Migration hoặc sử dụng `@AutoMigration`. |
| Crash: `IllegalStateException: Cannot access database on the main thread` | Gọi trực tiếp hàm DAO blocking trên Main Thread | Đổi hàm DAO thành `suspend fun` hoặc trả về `Flow<T>`, Room sẽ tự động điều phối xuống I/O Dispatcher nội bộ. |
| UI liên tục bị giật lag / giật khung hình mỗi khi có dữ liệu mới ghi vào DB | Thiếu `@Index` trên các cột Foreign Key hoặc cột `WHERE`, khiến mỗi lần re-query SQLite phải quét toàn bộ bảng | Thêm `indices = [Index("customerId")]` trong `@Entity`. |
| Crash: `SQLiteConstraintException: FOREIGN KEY constraint failed` | Chèn một `OrderItemEntity` với `parentOrderId` không tồn tại trong bảng `orders` | Luôn sử dụng `@Transaction` để chèn Parent Order trước, sau đó mới chèn Child Items. |

---

## 7. Tổng kết

Room Persistence Library kết hợp Kotlin Flow và Write-Ahead Logging tạo nên một giải pháp lưu trữ dữ liệu cục bộ **Phản ứng - Nhất quán - An toàn tuyệt đối**. Bằng cách hiểu rõ cơ chế của `InvalidationTracker`, giao dịch nguyên tử `@Transaction` và xây dựng chiến lược Migration bài bản, bạn đảm bảo ứng dụng Android luôn vận hành ổn định và bền vững trước mọi tình huống trong thực tế sản xuất.

---

*Chúc mừng bạn đã hoàn thành toàn bộ chương trình đào tạo chuẩn mực gồm 21 bài học chuyên sâu về Android Modern Development!*  
*Quay lại danh mục chính: [README](../README.md)*
