# Bài 09 — Paging 3 + Flow: Phân Trang Dữ Liệu Lớn & Offline Cache

> **Module:** 2 — Architecture & ViewModel  
> **Prerequisite:** [Bài 08 — Repository Pattern với Flow: Offline-First](08-repository-pattern-flow.md)  
> **Official Docs:**
> - [Paging 3 Library Overview — Android Developers](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)
> - [Paging with Jetpack Compose](https://developer.android.com/reference/kotlin/androidx/paging/compose/package-summary)
> - [Page from network and database (RemoteMediator)](https://developer.android.com/topic/libraries/architecture/paging/v3-network-db)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Paging 3** là thư viện chính thức thuộc bộ công cụ Android Jetpack, được thiết kế chuyên biệt để tải và hiển thị các tập dữ liệu có kích thước lớn (hàng nghìn đến hàng triệu bản ghi) theo từng trang (chunks/pages), giúp tiết kiệm băng thông mạng và tối ưu hóa bộ nhớ RAM của thiết bị di động.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        PAGING 3 ARCHITECTURE                           │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Repository Layer:                                              │   │
│   │   Pager(PagingConfig, RemoteMediator, PagingSourceFactory)     │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Trả về Flow<PagingData<T>>         │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ ViewModel Layer:                                               │   │
│   │   val pagingFlow = repo.getPagingStream().cachedIn(scope)      │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Thu thập PagingData                │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ UI Layer (Jetpack Compose):                                    │   │
│   │   val items = viewModel.pagingFlow.collectAsLazyPagingItems()  │   │
│   │   LazyColumn / LazyVerticalGrid                                │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Các thành phần cốt lõi của Paging 3

#### 1. `PagingSource<Key, Value>`
- Thành phần dữ liệu cấp thấp định nghĩa cách thức tải từng trang từ một nguồn dữ liệu cụ thể (ví dụ: gọi API lấy trang số $K$, hoặc truy vấn SQLite Room với Limit/Offset).
- Hàm cốt lõi: `suspend fun load(params: LoadParams<Key>): LoadResult<Key, Value>`.

#### 2. `RemoteMediator<Key, Value>`
- Cầu nối cao cấp giữa **Remote API (Mạng)** và **Local Database (Room)**.
- Đảm nhận việc tải trang từ mạng và lưu vào Room DB khi người dùng cuộn đến các ranh giới (đầu danh sách hoặc cuối danh sách).

#### 3. `Pager` & `PagingData<T>`
- `Pager`: Bộ khởi tạo tiếp nhận cấu hình phân trang (`PagingConfig`) và sinh ra một luồng phản ứng `Flow<PagingData<T>>`.
- `PagingData<T>`: Một container snapshot chứa các trang dữ liệu đã tải, đại diện cho tập dữ liệu có thể cuộn được trong bộ nhớ.

#### 4. `LoadState` & `LoadType`
- **`LoadType`**: Xác định hướng tải dữ liệu:
  - `REFRESH`: Tải lại toàn bộ dữ liệu từ trang đầu tiên (khi mở app hoặc kéo refresh).
  - `APPEND`: Tải trang tiếp theo khi người dùng cuộn xuống đáy danh sách.
  - `PREPEND`: Tải trang trước đó khi người dùng cuộn ngược lên đầu danh sách.
- **`LoadState`**: Trạng thái hiện tại của từng `LoadType`:
  - `LoadState.Loading`: Đang thực hiện request tải dữ liệu.
  - `LoadState.NotLoading`: Không tải, trạng thái nghỉ (có cờ `endOfPaginationReached`).
  - `LoadState.Error`: Tải thất bại, chứa đối tượng `error: Throwable`.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Bản chất của toán tử `cachedIn(viewModelScope)`

Một trong những quy tắc nghiêm ngặt nhất của Paging 3 là: **Bắt buộc phải gọi `.cachedIn(viewModelScope)` trên luồng `PagingData` trong ViewModel**.

```
                CƠ CHẾ HOẠT ĐỘNG CỦA CACHEDIN(VIEWMODELSCOPE)

Khi người dùng xoay màn hình (Configuration Change):

TRƯỜNG HỢP 1: KHÔNG CÓ cachedIn() (SAI LẦM):
Recomposition / Re-collect
       │
       ▼
Tạo ra một PagingData Stream hoàn toàn mới!
       │
       ▼
Toàn bộ danh sách bị tải lại từ trang 1 ──► MẤT VỊ TRÍ CUỘN, GIẬT LAG GIAO DIỆN!


TRƯỜNG HỢP 2: CÓ cachedIn(viewModelScope) (CHUẨN MỰC):
Recomposition / Re-collect
       │
       ▼
Multicast chia sẻ lại PagingData Snapshot đã có trong bộ nhớ cache của ViewModel!
       │
       ▼
Giao diện render lại tức thì trong 0ms ──► GIỮ NGUYÊN VỊ TRÍ CUỘN CỦA USER!
```

- `cachedIn()` đóng vai trò là một toán tử multicast (tương tự như `shareIn` với `Replay = 1`).
- Nó gắn kết vòng đời của các trang dữ liệu đã tải với `viewModelScope`. Dù Composable có bị hủy và tái tạo bao nhiêu lần, dữ liệu phân trang vẫn được giữ nguyên vẹn trong RAM!

---

### 2.2 Kiến trúc RemoteMediator: Phối hợp Mạng và Cơ sở dữ liệu

Khi xây dựng ứng dụng Offline-First có phân trang, `RemoteMediator` là giải pháp tối thượng:

```
┌────────────────────────────────────────────────────────────────────────┐
│                     REMOTEMEDIATOR WORKFLOW PIPELINE                   │
│                                                                        │
│   UI (Compose LazyColumn)                                              │
│         │                                                              │
│         ▼ Đọc dữ liệu cục bộ                                           │
│   Room Database (Local SSOT) ──PagingSource──► Hiển thị lên UI         │
│         ▲                                                              │
│         │                                                              │
│   Người dùng cuộn đến cuối trang...                                    │
│         │                                                              │
│         ▼ Kích hoạt callback                                           │
│   RemoteMediator.load(LoadType.APPEND)                                 │
│         │                                                              │
│         ├──► 1. Đọc RemoteKey từ Room để biết trang tiếp theo (ví dụ p=3)│
│         │                                                              │
│         ├──► 2. Gọi Retrofit API lấy dữ liệu trang 3                    │
│         │                                                              │
│         └──► 3. Lưu dữ liệu trang 3 + RemoteKey mới vào Room DB         │
│                     │                                                  │
│                     ▼                                                  │
│             Room tự động thông báo PagingSource phát data mới lên UI!  │
└────────────────────────────────────────────────────────────────────────┘
```

#### Tại sao cần bảng `RemoteKeys` trong SQLite?
Vì danh sách dữ liệu trong DB có thể bị chèn/xóa hoặc người dùng cuộn ngẫu nhiên, ta không thể đoán trước được trang tiếp theo cần fetch từ API là gì. Do đó, ta tạo một bảng SQLite phụ tên là `remote_keys` để lưu trữ thông tin `prevKey` và `nextKey` tương ứng với từng item!

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi kinh hoàng của việc Tự Code Phân Trang (Manual Pagination)

Trước khi có Paging 3, các lập trình viên phải tự quản lý phân trang với `RecyclerView.OnScrollListener` hoặc tính toán index trong Compose:

```
CÁC THÁCH THỨC CHÍ MẠNG KHI TỰ CODE PHÂN TRANG:
1. Tràn bộ nhớ (OutOfMemoryError):
   - Cứ mỗi lần cuộn xuống, ta lại add thêm items vào List: list = list + newPage.
   - Khi người dùng lướt Facebook/TikTok hàng giờ, danh sách lên tới 50,000 ảnh Bitmap -> CRASH OOM!

2. Race Conditions & Duplicate Requests:
   - Người dùng vẩy tay cuộn cực nhanh -> Bắn liên tiếp 4 request lấy page 2, 3, 4, 5 cùng lúc.
   - Các trang về sai thứ tự khiến danh sách bị lộn xộn hoặc duplicate dữ liệu.

3. Xử lý trạng thái tải phân mảnh:
   - Tự viết cờ: isLoading, isLastPage, isError, isRefreshing. Code ViewModel trở thành một mớ hỗn độn (Spaghetti code).
```

#### Paging 3 giải quyết triệt để:
- **Tự động giải phóng trang cũ:** Khi người dùng cuộn quá xa, Paging 3 tự động dọn dẹp các trang ở xa khỏi RAM để tránh tràn bộ nhớ.
- **Tự động hóa chống trùng lặp (Built-in Deduplication):** Không bao giờ bắn request trùng trang.
- **Trạng thái chuẩn hóa tuyệt đối (`CombinedLoadStates`):** Phân định rõ ràng từng trạng thái `Loading`, `Error`, `NotLoading` cho từng vị trí (Đầu trang, Giữa trang, Cuối trang).

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Bảng so sánh: Chỉ dùng `PagingSource` vs Dùng `RemoteMediator`

| Tiêu chí | Network-Only Paging (`PagingSource`) | Offline-Cache Paging (`RemoteMediator`) |
|---|---|---|
| **Nguồn dữ liệu** | Chỉ lấy trực tiếp từ Retrofit API. | Kết hợp Room Database + Retrofit API. |
| **Khả năng Offline** | ❌ Mất mạng là danh sách trống trơn hoặc báo lỗi. | ✅ Hoạt động offline hoàn hảo, đọc dữ liệu từ Room DB. |
| **Độ phức tạp mã nguồn** | Thấp, chỉ cần viết 1 class kế thừa `PagingSource`. | Cao hơn, cần bảng `RemoteKeys` và quản lý transaction Room. |
| **Use Cases phù hợp** | Tính năng Tìm kiếm (Search), dữ liệu biến động tức thời. | Bảng tin (News Feed), Sản phẩm (E-commerce), Lịch sử đơn hàng. |

---

### 4.2 Các nguyên tắc vàng trong Jetpack Compose với Paging 3

1. **Luôn sử dụng Stable Keys cho items trong LazyColumn:**
   ```kotlin
   // CHUẨN MỰC:
   LazyColumn {
       items(
           count = pagingItems.itemCount,
           key = pagingItems.itemKey { it.id } // Bắt buộc dùng itemKey ổn định
       ) { index ->
           val item = pagingItems[index]
           if (item != null) {
               PhotoCard(photo = item)
           }
       }
   }
   ```

2. **Phân tách giao diện theo từng loại `LoadState`:**
   - Khi `loadState.refresh is LoadState.Loading`: Hiển thị Progress Indicator toàn màn hình.
   - Khi `loadState.append is LoadState.Loading`: Hiển thị Progress Indicator nhỏ ở **chân trang (Footer Item)**.
   - Khi `loadState.append is LoadState.Error`: Hiển thị nút "Bấm để thử lại" ở chân trang.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng ứng dụng **Thư viện ảnh vô tận (Infinite Unsplash Photo Feed)** hỗ trợ Offline Caching với `RemoteMediator`, Room và Jetpack Compose.

### Bước 1: Khai báo Room Entities & RemoteKey Entity

```kotlin
// 1. Photo Entity
@Entity(tableName = "photos")
data class PhotoEntity(
    @PrimaryKey val id: String,
    val description: String?,
    val imageUrl: String,
    val photographer: String,
    val likes: Int
)

// 2. RemoteKeys Entity: Lưu con trỏ trang phục vụ phân trang tuần tự
@Entity(tableName = "photo_remote_keys")
data class PhotoRemoteKeys(
    @PrimaryKey val photoId: String,
    val prevKey: Int?,
    val nextKey: Int?
)

// 3. DAOs
@Dao
interface PhotoDao {
    // Trả về PagingSource do Room tự động sinh mã
    @Query("SELECT * FROM photos")
    fun getPagingSource(): PagingSource<Int, PhotoEntity>

    @Upsert
    suspend fun upsertAll(photos: List<PhotoEntity>)

    @Query("DELETE FROM photos")
    suspend fun clearAllPhotos()
}

@Dao
interface RemoteKeysDao {
    @Query("SELECT * FROM photo_remote_keys WHERE photoId = :id")
    suspend fun getRemoteKeysForPhoto(id: String): PhotoRemoteKeys?

    @Upsert
    suspend fun upsertAll(remoteKeys: List<PhotoRemoteKeys>)

    @Query("DELETE FROM photo_remote_keys")
    suspend fun clearRemoteKeys()
}
```

---

### Bước 2: Triển khai `PhotoRemoteMediator` (Cầu nối Network + Room)

```kotlin
@OptIn(ExperimentalPagingApi::class)
class PhotoRemoteMediator(
    private val database: AppDatabase,
    private val apiService: UnsplashApiService
) : RemoteMediator<Int, PhotoEntity>() {

    private val photoDao = database.photoDao()
    private val remoteKeysDao = database.remoteKeysDao()

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, PhotoEntity>
    ): MediatorResult {
        return try {
            // 1. Xác định số trang (page) cần tải từ API dựa trên LoadType
            val page = when (loadType) {
                LoadType.REFRESH -> {
                    val remoteKeys = getRemoteKeyClosestToCurrentPosition(state)
                    remoteKeys?.nextKey?.minus(1) ?: 1
                }
                LoadType.PREPEND -> {
                    val remoteKeys = getRemoteKeyForFirstItem(state)
                    val prevKey = remoteKeys?.prevKey
                        ?: return MediatorResult.Success(endOfPaginationReached = remoteKeys != null)
                    prevKey
                }
                LoadType.APPEND -> {
                    val remoteKeys = getRemoteKeyForLastItem(state)
                    val nextKey = remoteKeys?.nextKey
                        ?: return MediatorResult.Success(endOfPaginationReached = remoteKeys != null)
                    nextKey
                }
            }

            // 2. Gọi API lấy dữ liệu trang
            val response = apiService.getPhotos(page = page, pageSize = state.config.pageSize)
            val endOfPaginationReached = response.isEmpty()

            // 3. Lưu dữ liệu vào Room trong 1 Transaction nguyên tử
            database.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    remoteKeysDao.clearRemoteKeys()
                    photoDao.clearAllPhotos()
                }

                val prevKey = if (page == 1) null else page - 1
                val nextKey = if (endOfPaginationReached) null else page + 1
                
                val keys = response.map { photo ->
                    PhotoRemoteKeys(photoId = photo.id, prevKey = prevKey, nextKey = nextKey)
                }

                val entities = response.map { photo ->
                    PhotoEntity(photo.id, photo.description, photo.url, photo.photographer, photo.likes)
                }

                remoteKeysDao.upsertAll(keys)
                photoDao.upsertAll(entities)
            }

            MediatorResult.Success(endOfPaginationReached = endOfPaginationReached)
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }

    private suspend fun getRemoteKeyForLastItem(state: PagingState<Int, PhotoEntity>): PhotoRemoteKeys? {
        return state.pages.lastOrNull { it.data.isNotEmpty() }?.data?.lastOrNull()?.let { photo ->
            remoteKeysDao.getRemoteKeysForPhoto(photo.id)
        }
    }

    private suspend fun getRemoteKeyForFirstItem(state: PagingState<Int, PhotoEntity>): PhotoRemoteKeys? {
        return state.pages.firstOrNull { it.data.isNotEmpty() }?.data?.firstOrNull()?.let { photo ->
            remoteKeysDao.getRemoteKeysForPhoto(photo.id)
        }
    }

    private suspend fun getRemoteKeyClosestToCurrentPosition(state: PagingState<Int, PhotoEntity>): PhotoRemoteKeys? {
        return state.anchorPosition?.let { position ->
            state.closestItemToPosition(position)?.id?.let { id ->
                remoteKeysDao.getRemoteKeysForPhoto(id)
            }
        }
    }
}
```

---

### Bước 3: Repository & ViewModel (`Pager` + `cachedIn`)

```kotlin
// 1. Repository
class PhotoRepository(
    private val database: AppDatabase,
    private val apiService: UnsplashApiService
) {
    @OptIn(ExperimentalPagingApi::class)
    fun getPhotosPagingStream(): Flow<PagingData<PhotoEntity>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                prefetchDistance = 5,
                enablePlaceholders = false
            ),
            remoteMediator = PhotoRemoteMediator(database, apiService),
            pagingSourceFactory = { database.photoDao().getPagingSource() }
        ).flow
    }
}

// 2. ViewModel
class PhotoViewModel(
    photoRepository: PhotoRepository
) : ViewModel() {

    // QUY TẮC BẮT BUỘC: Gọi .cachedIn(viewModelScope)
    val photoPagingFlow: Flow<PagingData<PhotoEntity>> = photoRepository
        .getPhotosPagingStream()
        .cachedIn(viewModelScope)
}
```

---

### Bước 4: UI Layer (Jetpack Compose với `LazyPagingItems`)

```kotlin
@Composable
fun PhotoListScreen(
    viewModel: PhotoViewModel,
    modifier: Modifier = Modifier
) {
    // Thu thập luồng phân trang trong Compose
    val photos = viewModel.photoPagingFlow.collectAsLazyPagingItems()

    Scaffold(
        topBar = { TopAppBar(title = { Text("Thư viện ảnh Paging 3") }) }
    ) { padding ->
        Box(modifier = modifier.fillMaxSize().padding(padding)) {
            // Xử lý Loading toàn màn hình khi mở trang đầu tiên
            if (photos.loadState.refresh is LoadState.Loading) {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            } else if (photos.loadState.refresh is LoadState.Error) {
                val error = (photos.loadState.refresh as LoadState.Error).error
                Column(
                    modifier = Modifier.align(Alignment.Center).padding(16.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Text(text = "Lỗi tải dữ liệu: ${error.localizedMessage}", color = MaterialTheme.colorScheme.error)
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { photos.retry() }) {
                        Text("Thử lại")
                    }
                }
            } else {
                LazyColumn(
                    modifier = Modifier.fillMaxSize().padding(horizontal = 16.dp),
                    verticalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    // Danh sách ảnh
                    items(
                        count = photos.itemCount,
                        key = photos.itemKey { it.id }
                    ) { index ->
                        val photo = photos[index]
                        if (photo != null) {
                            PhotoCardItem(photo)
                        }
                    }

                    // Xử lý trạng thái tải ở cuối trang (Append / Load More)
                    when (val appendState = photos.loadState.append) {
                        is LoadState.Loading -> {
                            item {
                                Box(modifier = Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                                    CircularProgressIndicator(modifier = Modifier.size(32.dp))
                                }
                            }
                        }
                        is LoadState.Error -> {
                            item {
                                Row(
                                    modifier = Modifier.fillMaxWidth().padding(16.dp),
                                    horizontalArrangement = Arrangement.SpaceBetween,
                                    verticalAlignment = Alignment.CenterVertically
                                ) {
                                    Text("Lỗi tải thêm ảnh", color = MaterialTheme.colorScheme.error)
                                    Button(onClick = { photos.retry() }) {
                                        Text("Thử lại")
                                    }
                                }
                            }
                        }
                        is LoadState.NotLoading -> Unit
                    }
                }
            }
        }
    }
}

@Composable
fun PhotoCardItem(photo: PhotoEntity) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = photo.photographer, style = MaterialTheme.typography.titleMedium)
            Text(text = photo.description ?: "Không có mô tả", style = MaterialTheme.typography.bodyMedium)
            Spacer(modifier = Modifier.height(4.dp))
            Text(text = "❤️ ${photo.likes} lượt thích", style = MaterialTheme.typography.labelSmall)
        }
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Làm thế nào để thêm/xóa/sửa (CRUD) một item đơn lẻ trong Paging 3 mà không cần reload lại toàn bộ dữ liệu từ đầu?
**Trả lời chuẩn bản chất:**
Trong Paging 3 kết hợp Room (`RemoteMediator`), bạn **không trực tiếp chỉnh sửa đối tượng `PagingData`**. Thay vào đó, bạn chỉ cần thực hiện thao tác xóa/sửa trực tiếp vào **Room Database** (ví dụ gọi `photoDao.deleteById(id)` hoặc `photoDao.updateLikes(id, newLikes)`). Khi bản ghi trong SQLite thay đổi, `PagingSource` của Room sẽ tự động phát hiện sự thay đổi và âm thầm cập nhật trang chứa item đó trên giao diện người dùng mà không cần reset lại scroll position!

#### Q2: Sự khác nhau giữa `pageSize` và `prefetchDistance` trong `PagingConfig`?
**Trả lời chuẩn bản chất:**
- **`pageSize`**: Số lượng bản ghi được yêu cầu tải trong mỗi trang từ `PagingSource` (thông thường là 20-50 items).
- **`prefetchDistance`**: Khoảng cách đón đầu (tính bằng số lượng items còn lại trước khi cuộn đến đáy). Ví dụ nếu `prefetchDistance = 5`, khi người dùng cuộn đến vị trí cách phần tử cuối cùng 5 items, Paging 3 sẽ lập tức âm thầm kích hoạt tải trang tiếp theo ở chế độ nền. Điều này tạo ra trải nghiệm "cuộn vô tận mượt mà" vì dữ liệu đã có sẵn trước khi người dùng chạm tới đáy!

#### Q3: Tại sao gọi `PagingData.filter` hoặc `PagingData.map` trong ViewModel có thể gây lãng phí hiệu năng?
**Trả lời chuẩn bản chất:**
Nếu bạn thực hiện các phép biến đổi tính toán nặng trên từng phần tử của `PagingData` trong ViewModel, thao tác này sẽ chạy lại mỗi khi có trang mới được phát ra. Cách làm chuẩn mực của Senior Architect là thực hiện lọc (`filter`) và sắp xếp (`ORDER BY`) ngay từ câu truy vấn SQL của Room DAO hoặc từ Backend API, thay vì xử lý thủ công trên bộ nhớ máy khách.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `IllegalArgumentException: Key must be unique`** | Cung cấp ID trùng lặp trong hàm `items(key = { ... })` của Compose LazyColumn. | Đảm bảo trường khóa chính (`id`) của API và Database là duy nhất tuyệt đối. |
| **Danh sách bị cuộn ngược về đầu trang sau khi xoay màn hình** | Quên gọi `.cachedIn(viewModelScope)` trong ViewModel. | Bổ sung `.cachedIn(viewModelScope)` vào chuỗi Flow của ViewModel. |
| **Vòng lặp tải liên tục không dừng (Infinite Paging Loop)** | Trả về `endOfPaginationReached = false` trong `RemoteMediator` dù API đã hết dữ liệu (trả về danh sách rỗng). | Kiểm tra `val endOfPaginationReached = response.isEmpty()` và trả về `MediatorResult.Success(endOfPaginationReached)`. |

---

*Bài trước: [08 — Repository Pattern với Flow: Offline-First](08-repository-pattern-flow.md)*  
*Module tiếp theo: [Module 3 — Jetpack Compose Foundation](../module-3-compose-foundation/10-composable-functions.md)*
