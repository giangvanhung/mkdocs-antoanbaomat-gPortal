# 05 — Kịch bản kiểm thử

> **Nguyên tắc:** biên dịch sạch chỉ chứng minh mã nguồn hợp lệ về cú pháp và
> kiểu dữ liệu. Nó **không nói gì** về việc chốt chặn có thật sự chặn hay
> không. Thứ bảo vệ khỏi việc tin nhầm không phải là đọc kỹ code, mà là **vòng
> kiểm tra khách quan**: chạy thật và nhìn kết quả.

---

## 0. Chuẩn bị bắt buộc

### a. Chạy migration

```
Database/DbUpdate.sql
```

**Bắt buộc, không phải tùy chọn.** Mô hình EF hiện có 5 cột và 1 bảng mà CSDL
chưa có. Thiếu bất kỳ cột nào thì câu `SELECT` sinh ra bởi EF sẽ lỗi, và
`CacheHelper.GetCurrentPortalSettings()` hỏng → **toàn bộ portal sập**, không
riêng gì tính năng mới.

Kiểm tra sau khi chạy:

```sql
SELECT MigrationId, AppliedOnUtc FROM dbo.gp_DbMigrations ORDER BY MigrationId;
-- Phải có đủ: 000 → 007
```

### b. Build lại giao diện admin

```powershell
.\build-gclient.ps1 -App gPortalAdmin
```

Không build lại thì các ô cấu hình mới **không hiện ra** trong màn hình quản
trị, dù backend đã sẵn sàng.

---

## 1. Chính sách mật khẩu định kỳ

**Cấu hình:** Bảo mật → Thời gian yêu cầu đổi = `60`, Thời gian hợp lệ = `90`

Dùng SQL để "tua ngược thời gian" cho một tài khoản thử:

```sql
-- Vùng ÂN HẠN: 70 ngày > 60, chưa tới 90
UPDATE dbo.gp_Users
SET PasswordChangedUtc = DATEADD(DAY, -70, GETUTCDATE())
WHERE UserName = 'gianghungtest';
```

| Bước | Kết quả phải thấy |
|---|---|
| Đăng nhập | Vào trang đổi mật khẩu, khung **xanh nhạt** (`alert-info`) |
| Đọc thông báo | Có **số ngày còn lại** trước khi bị khóa (20 ngày) |
| Bấm "Để sau" | ✅ **Vào được hệ thống bình thường** |
| Mở các chức năng khác | ✅ Dùng được hết |

```sql
-- Vùng KHÓA: 95 ngày > 90
UPDATE dbo.gp_Users
SET PasswordChangedUtc = DATEADD(DAY, -95, GETUTCDATE())
WHERE UserName = 'gianghungtest';
```

| Bước | Kết quả phải thấy |
|---|---|
| Đăng nhập | Vào trang đổi mật khẩu, khung **vàng/đỏ** |
| Nút "Để sau" | ❌ **Không xuất hiện** |
| Gõ thẳng URL `/apps/home.aspx` | ❌ Bị đá về trang đổi mật khẩu |
| Đổi mật khẩu thành công | ✅ Vào được ngay — khóa **tự tan**, không cần thao tác gì thêm |

**Thử lỗ hổng đã vá** (đường dẫn chứa chuỗi cho qua):

```
/api/report/account/login-history
```
Phải **bị chặn**. Nếu vào được thì `DuongDanRequest` đang so sai.

**Thử ràng buộc cấu hình** — điền một ô, để trống ô kia → phải bị từ chối lưu
với thông báo rõ ràng. Thử cả bằng giao diện **và** bằng gọi thẳng API (Postman)
để chắc chắn chốt ở máy chủ đang hoạt động, không chỉ validator ExtJS.

---

## 2. Khóa tài khoản theo cửa sổ thời gian

**Cấu hình:** Số lần sai tối đa = `3`, Khoảng tính số lần sai = `2` phút

```sql
-- Xem trạng thái bộ đếm bất cứ lúc nào
SELECT UserName, AccessFailedCount, LastFailedAttemptUtc, LockoutEndDateUtc
FROM dbo.gp_Users WHERE UserName = 'gianghungtest';
```

### Kịch bản A — cửa sổ phải đặt lại bộ đếm

| Bước | `AccessFailedCount` sau bước đó |
|---|---|
| Nhập sai lần 1 | 1 |
| Nhập sai lần 2 | 2 |
| **Chờ hơn 2 phút** | 2 (chưa đổi — bộ đếm chỉ được xét khi có lần sai mới) |
| Nhập sai lần 3 | **1** ← đã đặt lại |
| Kết quả | ✅ **Không** bị khóa |

### Kịch bản B — sai liên tiếp phải khóa

| Bước | Kết quả |
|---|---|
| Nhập sai 3 lần liên tiếp (dưới 2 phút) | ❌ Tài khoản bị khóa |
| `LockoutEndDateUtc` | Có giá trị trong tương lai |

### Kịch bản C — lỗi đếm hai lần đã vá

Làm lại kịch bản B **qua đường `MobileLogin`** (API di động).

Phải khóa ở **đúng lần thứ 3**, không phải lần thứ 2. Nếu khóa ở lần thứ 2 thì
`return null;` trong nhánh `AccessFailedAsync` đã bị mất.

---

## 3. Thời gian chờ của phiên làm việc

**Cấu hình:** Thời gian chờ = `2` phút, Cảnh báo trước = `1` phút

> Cảnh báo phải **nhỏ hơn hẳn** thời gian chờ. Để trống cũng được — khi đó
> không có hộp cảnh báo nhưng vẫn phải tự đăng xuất.

### Kịch bản A — đồng hồ MÁY CHỦ (không dính dáng gì tới JavaScript)

Đây là phép thử sạch nhất, vì nó loại bỏ hoàn toàn script client và việc ExtJS
gọi AJAX ở nền.

| Bước | Kết quả phải thấy |
|---|---|
| Đăng nhập | Vào được |
| **Đóng hết tab trình duyệt** | — |
| Chờ **3 phút** | — |
| Mở lại `/apps/home.aspx` | ❌ Bị đá về trang đăng nhập |
| Trang đăng nhập | Hiện khung **vàng** kèm lý do hết phiên |

> **Không cần mở trang để chờ.** Máy chủ không có job nền — nó quyết định
> **tại thời điểm request kế tiếp**. Không có request thì không có gì xảy ra,
> và cũng không cần, vì lúc đó không ai đang dùng hệ thống.

### Kịch bản B — đồng hồ CLIENT

| Bước | Kết quả phải thấy |
|---|---|
| Mở `/admin/portal.aspx`, không đụng chuột/phím | — |
| Sau **1 phút** | Hộp cảnh báo hiện, có **đếm ngược giây** |
| Rê chuột trong lúc hộp đang hiện | Hộp **KHÔNG** biến mất (cố ý) |
| Bấm "Tiếp tục làm việc" | ✅ Hộp đóng, làm việc tiếp bình thường |
| Không bấm gì, chờ hết giờ | ❌ Tự chuyển tới trang đăng nhập kèm thông báo |

### Kịch bản C — chứng minh vì sao cần đồng hồ client

| Bước | Kết quả |
|---|---|
| Mở trang admin có ứng dụng ExtJS, để yên | ExtJS vẫn gọi AJAX ở nền |
| Nhìn tab Network | Có request đều đặn |
| Nếu **chỉ có** đồng hồ máy chủ | Phiên sẽ **không bao giờ** hết giờ |
| Nhờ có đồng hồ client | Vẫn hết giờ đúng hạn |

Đây là toàn bộ lý do tồn tại của `gp-session-timeout.js`.

### Nếu không thấy gì xảy ra

Xem mục 7 của [03-thoi-gian-cho-phien.md](03-thoi-gian-cho-phien.md) — có quy
trình chẩn đoán từng bước (`gpSessionTimeoutDaChay` trong Console → tab Network
→ câu SQL kiểm tra cấu hình).

---

## 4. Giới hạn địa chỉ mạng quản trị

⚠️ **Nên thử trên môi trường phát triển trước.** Đây là tính năng có thể khóa
chính bạn ra ngoài. Trước khi thử trên máy chủ thật, hãy chắc chắn bạn biết
cách dùng công tắc `gp:AdminIpRestrictionDisabled` trong `Web.config`.

**Chuẩn bị:** Vào tab **"Địa chỉ quản trị"**, đọc con số ở góc phải thanh công
cụ: *"Địa chỉ của bạn: ..."*. Ghi lại chính xác con số đó.

> Trên localhost con số này thường là `::1` chứ không phải `127.0.0.1`. Cứ dùng
> đúng con số hiện ra.

### Kịch bản A — chốt chống tự khóa

| Bước | Kết quả phải thấy |
|---|---|
| Thêm quy tắc `203.0.113.5` (một địa chỉ **không phải** của bạn) | ❌ **Từ chối lưu**, hộp thoại giải thích rõ |
| Đọc thông báo | Có địa chỉ hiện tại của bạn + việc phải làm tiếp |
| Trạng thái sau đó | Vẫn vào được hệ thống bình thường |

Đây là kịch bản quan trọng nhất. Nếu nó lưu được thì chốt chống tự khóa hỏng.

### Kịch bản B — thêm đúng địa chỉ của mình

| Bước | Kết quả |
|---|---|
| Thêm quy tắc = **địa chỉ của bạn** | ✅ Lưu thành công |
| Tải lại `/admin/portal.aspx` | ✅ Vẫn vào bình thường |
| Bây giờ thêm `203.0.113.5` | ✅ **Lưu được** (vì đã có dòng bảo vệ bạn) |

### Kịch bản C — chặn thật

Cần một máy/thiết bị thứ hai ở địa chỉ khác (điện thoại dùng 4G là đủ):

| Bước | Kết quả |
|---|---|
| Từ máy thứ hai, mở `/admin/portal.aspx` | ❌ Trang 403, có hiện địa chỉ của máy đó |
| Từ máy thứ hai, đăng nhập bằng tài khoản **quản trị** | ❌ Về trang đăng nhập, khung đỏ giải thích lý do |
| Từ máy thứ hai, đăng nhập bằng tài khoản **thường** | ✅ Vào được (chính sách chỉ nhắm quản trị viên) |
| Từ máy thứ hai, xem trang công khai | ✅ Xem được |

### Kịch bản D — tắt tính năng

| Bước | Kết quả |
|---|---|
| Xóa dần cho tới dòng cuối cùng | ✅ Dòng cuối **xóa được** (đó là cách tắt) |
| Từ máy thứ hai | ✅ Vào lại được |

### Kịch bản E — kiểm cú pháp

Thử lưu các giá trị sau, tất cả phải **bị từ chối** kèm thông báo cụ thể:

| Giá trị | Vì sao sai |
|---|---|
| `192.168.1` | Thiếu đoạn — .NET sẽ hiểu nhầm thành `192.168.0.1` |
| `192.168.1.0/33` | Độ dài tiền tố vượt quá 32 bit |
| `192.168.1.50-192.168.1.1` | Đầu lớn hơn cuối → khoảng rỗng, không ai khớp |
| `abc.def` | Không phải địa chỉ |

Thử **cả bằng gọi thẳng API** (Postman) chứ không chỉ qua giao diện — validator
ExtJS chỉ là trải nghiệm.

### Kịch bản F — công tắc khẩn cấp

| Bước | Kết quả |
|---|---|
| Thêm `<add key="gp:AdminIpRestrictionDisabled" value="true" />` vào `Web.config` | Ứng dụng khởi động lại |
| Từ máy thứ hai | ✅ Vào lại được ngay |

---

## 5. Danh mục kiểm tra tổng

- [ ] `DbUpdate.sql` đã chạy, `gp_DbMigrations` có đủ 000 → 007
- [ ] `gPortalAdmin` đã build lại, ba tab hiện đủ ô cấu hình
- [ ] Mật khẩu: ân hạn cho qua, hết hạn thì chặn, đổi xong thì mở
- [ ] Mật khẩu: `/api/report/account/login-history` bị chặn
- [ ] Khóa tài khoản: cửa sổ đặt lại bộ đếm đúng
- [ ] Khóa tài khoản: `MobileLogin` khóa ở đúng lần thứ N
- [ ] Phiên: đóng hết trình duyệt, chờ, mở lại → bị đá ra
- [ ] Phiên: hộp cảnh báo hiện, nút "Tiếp tục làm việc" hoạt động
- [ ] Địa chỉ mạng: chốt chống tự khóa từ chối đúng
- [ ] Địa chỉ mạng: máy ngoài bị 403, người dùng thường không bị ảnh hưởng
- [ ] Địa chỉ mạng: công tắc `Web.config` mở lại được
- [ ] Mọi ràng buộc đều được thử **qua API trực tiếp**, không chỉ qua giao diện
