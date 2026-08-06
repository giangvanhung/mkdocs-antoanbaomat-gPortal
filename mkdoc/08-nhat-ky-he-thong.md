# 08 — Nhật ký hệ thống

## Yêu cầu nghiệp vụ

> (b) Nhật ký hệ thống được phân loại theo ít nhất **05 nhóm**:
> i. Nhật ký truy cập Phần mềm;
> ii. Nhật ký đăng nhập khi quản trị Phần mềm;
> iii. Nhật ký các lỗi phát sinh trong quá trình hoạt động;
> iv. Nhật ký quản lý tài khoản;
> v. Nhật ký thay đổi cấu hình Phần mềm.

---

## 1. Vì sao bốn tính năng trước chưa đủ

Bốn tính năng bảo mật trước đều là **chốt chặn** — chúng làm rất tốt việc *ngăn*
chuyện xấu. Nhưng một hệ thống bảo mật phải trả lời được **hai** loại câu hỏi:

| Câu hỏi | Ai trả lời |
|---|---|
| "Ngăn được không?" | Chốt chặn |
| "Đã từng có ai thử chưa?" | **Nhật ký** |

Trước đợt này, sự kiện đáng ghi nhất — **tài khoản bị khóa do đăng nhập sai** —
không để lại dấu vết nào tồn tại quá vài phút. Dấu vết duy nhất là cột
`AccessFailedCount`, mà cột đó bị **ghi đè** ngay khi đăng nhập đúng hoặc khi
cửa sổ thời gian hết hạn. Câu hỏi *"tháng trước có ai dò mật khẩu tài khoản
admin không?"* không trả lời được.

---

## 2. Vì sao là bảng CSDL chứ không phải file log4net

Dự án đã có log4net ghi ra `App_Data\gPortal.txt`. Nhưng đó là **văn bản tự do**.

Yêu cầu đòi nhật ký phải được **phân loại** theo năm nhóm và **tra cứu được**.
Một file văn bản thì:

- không lọc theo nhóm,
- không phân trang,
- không tìm theo người dùng hay khoảng thời gian.

Muốn làm những việc đó trên file thì phải tự viết bộ phân tích cú pháp cho
chính định dạng mình vừa in ra — và nó sẽ hỏng ngay lần đầu có ai đổi mẫu in.

### log4net vẫn giữ nguyên, với vai trò khác

Nó là **lưới an toàn** cho chính bảng nhật ký. Khi `NhatKyHeThong` không ghi
được xuống CSDL, nó đổ ra file kèm tiền tố `NHATKY_ROI`:

``` title="Định dạng dòng dự phòng — gPortal.Framework/NhatKyHeThong.cs:186"
NHATKY_ROI | loai=QuanLyTaiKhoan | hanhDong=KHOA_TAI_KHOAN | thanhCong=False | ...
```

Tìm chuỗi đó trong file log là cách biết bảng nhật ký có đầy đủ hay không.

---

## 3. Ba luật bất di bất dịch của `NhatKyHeThong`

### Luật 1 — Không bao giờ được ném ngoại lệ

Nhật ký là **người quan sát**, không phải người tham gia. Một thao tác nghiệp
vụ hợp lệ không được phép thất bại chỉ vì ghi nhật ký hỏng.

Đảo lại thì việc thêm nhật ký — vốn để làm hệ thống an toàn hơn — lại trở thành
nguồn sự cố mới. Mọi lời gọi đều nằm trong `try/catch`.

### Luật 2 — Dùng kết nối riêng, không dùng `DbContext` của request

```csharp title="gPortal_Portal/gPortal.Framework/NhatKyHeThong.cs:88-91"
using (var db = new ApplicationDbContext())
{
    db.AuditLogs.Add(dong);
    db.SaveChanges();
}
```

Ta ghi lại cả những thao tác **thất bại**. Một thao tác thất bại có thể đã làm
hỏng context của request, hoặc sẽ bị cuộn lại. Ghi chung thì **đúng những dòng
nhật ký quan trọng nhất — dòng ghi lại thất bại — sẽ biến mất cùng với thất bại
đó**.

### Luật 3 — Hỏng thì phải kêu, không được im

Một hệ thống nhật ký âm thầm mất dữ liệu còn **tệ hơn không có nhật ký**, vì nó
tạo cảm giác an toàn giả.

---

## 4. Ghi SỰ KIỆN, không ghi REQUEST

Nhóm (i) "Nhật ký truy cập Phần mềm" **không** ghi mỗi lượt xem trang.

Lý do là số học: portal này phục vụ hàng nghìn request mỗi phút kể cả ảnh và
file css. Ghi từng request là thêm một câu `INSERT` vào từng request, và bảng sẽ
đạt hàng chục triệu dòng trong vài tuần. Đến lúc đó không ai tra cứu nổi nữa —
tức là **nhật ký mất tác dụng đúng vì đã ghi quá nhiều**.

Thay vào đó ghi các sự kiện thuộc **vòng đời truy cập**: đăng nhập thành công,
đăng nhập thất bại, đăng xuất, hết phiên, bị từ chối truy cập. Đó là những mốc
trả lời được *"ai đã vào hệ thống, khi nào, từ đâu"* — mục đích thật sự của
nhóm này.

---

## 5. Sự kiện nào vào nhóm nào

| Nhóm | Sự kiện | Mã hành động | Móc ở đâu |
|---|---|---|---|
| **i** Truy cập | Đăng nhập thành công (người dùng thường) | `DANG_NHAP` | `AccountController.RedirectAfterSignIn` |
| i | Đăng nhập sai mật khẩu | `DANG_NHAP_SAI_MAT_KHAU` | `IdentityConfig.AccessFailedAsync` |
| i | Tên đăng nhập không tồn tại | `DANG_NHAP_SAI_TEN` | `AccountController.Login` |
| i | Đăng nhập vào tài khoản đang bị khóa | `DANG_NHAP_TAI_KHOAN_BI_KHOA` | `AccountController.Login` |
| i | Đăng xuất | `DANG_XUAT` | `AccountController.LogOff` |
| i | Phiên bị đóng do hết giờ | `HET_PHIEN` | `SessionTimeoutGuard` |
| **ii** Quản trị | Mọi sự kiện trên, khi tài khoản là **quản trị viên** | (như trên) | `NhatKyHeThong.LoaiTheoVaiTro` |
| ii | Bị chặn vì địa chỉ mạng | `CHAN_DIA_CHI_QUAN_TRI` | `AdminIpGuard.Handle` |
| ii | Từ chối đăng nhập vì địa chỉ mạng | `TU_CHOI_DIA_CHI_DANG_NHAP` | `AccountController.ChanQuanTriSaiDiaChi` |
| **iii** Lỗi | Ngoại lệ không được bắt | `LOI_HE_THONG` | `Global.asax.Application_Error` |
| **iv** Tài khoản | Tài khoản bị khóa | `KHOA_TAI_KHOAN` | `IdentityConfig.AccessFailedAsync` |
| iv | Xóa tài khoản | `XOA_TAI_KHOAN` | `AdminController.DeleteUser` |
| iv | Gán vai trò | `GAN_VAI_TRO` | `AdminController.AddUserRoles` |
| **v** Cấu hình | Sửa cấu hình bảo mật | `SUA_CAU_HINH_BAO_MAT` | `AdminController.AddPortalSettings` |
| v | Thêm/sửa/xóa quy tắc địa chỉ | `*_DIA_CHI_QUAN_TRI` | `AdminController` |

### Một dòng hay hai dòng?

Chú ý cặp `DANG_NHAP_SAI_MAT_KHAU` (nhóm i/ii) và `KHOA_TAI_KHOAN` (nhóm iv).
Cùng một lần bấm nút của người dùng, nhưng **hai dòng nhật ký ở hai nhóm**:

- *sai mật khẩu* → một lượt truy cập hỏng
- *khóa tài khoản* → **trạng thái** tài khoản đã đổi

Gộp làm một thì người kiểm toán đi tìm các lần khóa tài khoản sẽ phải lọc trong
hàng nghìn dòng đăng nhập sai.

---

## 6. Đặt điểm móc ở đâu — nguyên tắc lặp lại

`DANG_NHAP_SAI_MAT_KHAU` ban đầu được viết trong `AccountController`, rồi
**chuyển xuống** `IdentityConfig.AccessFailedAsync`.

Lý do y hệt lý do cửa sổ đếm đã nằm ở đó: có **ba lối gọi** tới
`AccessFailedAsync` —

1. đăng nhập web (`AccountController.Login`)
2. đăng nhập di động (`AccountController.MobileLogin`)
3. bên trong `SignInManager.PasswordSignInAsync`

Ghi nhật ký ở controller thì phải chép ba bản, và **bản nào bị quên thì sẽ có
một đường đăng nhập không để lại vết** — đúng đường mà kẻ tấn công sẽ tìm ra
trước.

Cùng nguyên tắc với `RedirectAfterSignIn` (điểm hội tụ của bốn đường đăng nhập
web) và `AdminIpGuard.DuocPhep` (dùng chung giữa chốt chặn và chốt chống tự
khóa). Ba ví dụ, một nguyên tắc.

---

## 7. Một cái bẫy có thật: múi giờ

EF trả `DateTime` với `Kind = Unspecified`. `JavaScriptSerializer` mà MVC dùng
cho `Json()` sẽ gọi `ToUniversalTime()` lên giá trị đó — và với
`Kind = Unspecified`, .NET coi nó là giờ **địa phương**, trừ đi 7 tiếng ở Việt
Nam. Trình duyệt sau đó cộng lại 7 tiếng.

Kết quả: màn hình hiện đúng con số UTC nhưng **dán nhãn là giờ địa phương**.
Mọi mốc thời gian lệch 7 tiếng, và người đọc sẽ kết luận sai về **thứ tự các sự
kiện** — đúng thứ mà nhật ký sinh ra để trả lời.

Cách sửa dứt điểm: trả về chuỗi ISO-8601 có hậu tố `Z`.

```csharp title="gPortal_Portal/gPortal/Controllers/AdminController.cs:1310"
ThoiGianUtc = DateTime.SpecifyKind(x.ThoiGianUtc, DateTimeKind.Utc).ToString("o")
```

Nó nói rõ đây là giờ UTC, trình duyệt tự đổi sang giờ máy người xem, và **mở
JSON ra là đọc được ngay múi giờ** — không phải đoán.

---

## 8. Không có nút xóa, và đó là chủ ý

Màn hình nhật ký **không có** chức năng xóa. API cũng không có endpoint xóa.

Một nhật ký kiểm toán mà **người bị kiểm toán xóa được** thì không còn là bằng
chứng. Quản trị viên chính là đối tượng mà nhóm (ii) và nhóm (v) theo dõi — cho
họ nút xóa là tự phá bỏ mục đích của cả tính năng.

### Vậy dọn dữ liệu cũ thế nào?

Đây là việc của người quản trị CSDL, có kiểm soát, và **nên sao lưu trước**:

```sql title="Chạy trong SQL Server Management Studio"
-- Xem khối lượng trước khi quyết định
SELECT LoaiNhatKy, COUNT(*) AS SoDong,
       MIN(ThoiGianUtc) AS CuNhat, MAX(ThoiGianUtc) AS MoiNhat
FROM dbo.gp_AuditLogs
GROUP BY LoaiNhatKy;

-- Sao lưu trước khi xóa (đổi tên bảng theo ngày)
SELECT * INTO dbo.gp_AuditLogs_LuuTru_202608
FROM dbo.gp_AuditLogs
WHERE ThoiGianUtc < DATEADD(MONTH, -6, GETUTCDATE());

-- Chỉ xóa sau khi đã kiểm tra bảng lưu trữ có đủ dòng
DELETE FROM dbo.gp_AuditLogs
WHERE ThoiGianUtc < DATEADD(MONTH, -6, GETUTCDATE());
```

Thời gian lưu tối thiểu phụ thuộc quy định áp dụng cho hệ thống — hãy hỏi trước
khi xóa bất cứ thứ gì.

---

## 9. Ba chỉ mục, mỗi chỉ mục một câu hỏi

Bảng này **chỉ tăng, không giảm**. Không có chỉ mục thì đến tháng thứ ba mọi
truy vấn đều quét toàn bảng, và màn hình nhật ký sẽ treo **đúng lúc cần nó nhất
— lúc có sự cố, tức là lúc bảng dài nhất**.

| Chỉ mục | Trả lời câu hỏi |
|---|---|
| `IX_gp_AuditLogs_ThoiGian` | "xem 50 dòng gần nhất" (mặc định của màn hình) |
| `IX_gp_AuditLogs_LoaiThoiGian` | "lọc theo nhóm rồi sắp xếp" (thao tác hay dùng nhất) |
| `IX_gp_AuditLogs_ThanhCong` | "chỉ xem các sự kiện thất bại" |

Phân trang cũng chạy **phía máy chủ** (`start` / `limit`), với trần 500 dòng mỗi
lần gọi — giao diện không bao giờ gửi số lớn hơn, nhưng giao diện không phải là
thứ duy nhất gọi được API.

---

## 10. Giao diện

**gPortalAdmin → Nhật ký hệ thống**

| Thành phần | Ghi chú |
|---|---|
| Lọc theo nhóm | Danh sách lấy từ `/Admin/AuditLogTypes`, sinh từ enum — thêm nhóm thứ sáu thì ô này tự có |
| Lọc theo khoảng ngày | Ô "Đến" bao gồm **cả** ngày được chọn |
| Tìm theo từ khóa | Khớp người dùng, mô tả, IP, mã hành động |
| "Chỉ sự kiện thất bại" | Lối tắt tới thứ cần nhìn nhất |
| Dòng thất bại tô đỏ nhạt | Để mắt bắt được **cụm** sự kiện hỏng liên tiếp |
| Bấm đúp một dòng | Mở hộp chi tiết (giá trị cấu hình trước/sau, nội dung ngoại lệ) |

---

## 11. Sắp xếp lại màn hình cấu hình

Tab "Bảo mật" cũ là **một danh sách dọc dài** trộn lẫn: cách đăng nhập, quy tắc
mật khẩu, khóa tài khoản, thời gian chờ phiên, và **cấu hình SMTP**.

Đã tách thành các tab, mỗi tab trả lời một câu hỏi:

| Tab | Câu hỏi | Tài liệu tương ứng |
|---|---|---|
| Thiết lập chung | Trang này tên gì, dùng mẫu nào | — |
| **Mật khẩu** | Mật khẩu phải như thế nào, dùng được bao lâu | [01](01-mat-khau-dinh-ky.md) |
| **Đăng nhập** | Đăng nhập bằng gì, sai thì sao | [02](02-khoa-tai-khoan.md) |
| **Phiên làm việc** | Ngồi không bao lâu thì bị đóng phiên | [03](03-thoi-gian-cho-phien.md) |
| **Địa chỉ quản trị** | Ai được quản trị, từ đâu | [04](04-dia-chi-mang-quan-tri.md) |
| **Email (SMTP)** | Gửi thư qua đâu | — |

Hai điều đáng nói:

1. **SMTP bị tách khỏi "Bảo mật".** Nó là hạ tầng gửi thư, không phải chính
   sách bảo mật. Để lẫn thì danh sách tham số bảo mật dài ra mà không có lý do,
   và người đi tìm cấu hình SMTP không nghĩ tới việc mở tab Bảo mật.

2. **"Phiên làm việc" tách khỏi "Đăng nhập".** Ba tham số khóa tài khoản nói về
   đăng nhập **sai**; hai tham số phiên nói về một phiên **đang mở mà không có
   thao tác nào**. Để chung một danh sách thì rất dễ đọc nhầm "thời gian khóa
   tài khoản" thành "thời gian chờ của phiên" — hai con số đều tính bằng phút,
   nằm cạnh nhau, mà ý nghĩa hoàn toàn khác.

Mỗi tab giờ ánh xạ đúng **một** file tài liệu. Đó không phải trùng hợp: nếu một
tab cần hai file để giải thích thì nó đang gộp hai thứ không liên quan.

---

## 12. Nơi cần biết

| Việc | Tệp |
|---|---|
| Ghi nhật ký | `gPortal.Framework/NhatKyHeThong.cs` |
| Mô hình + enum 5 nhóm | `gPortal.Framework/Identity/IdentityModels.cs` (`gp_AuditLogs`, `LoaiNhatKy`) |
| Lấy địa chỉ IP (dùng chung với chốt chặn) | `gPortal.Framework/Security/DiaChiClient.cs` |
| API đọc + lọc | `AdminController` — vùng `#region "AuditLogs"` |
| Giao diện | `gPortalAdmin/app/view/Log/PortalLogs.js` |
| Model + store | `gPortalAdmin/app/model/mAuditLog.js`, `app/store/sAuditLog.js` |
| Điều hướng | `gPortalAdmin/app/store/NavigationTree.js` |
| Migration | `Database/DbUpdate.sql` — 008 |

---

## 13. Kiểm thử

Sau khi chạy migration 008 và build lại `gPortalAdmin`:

| Việc làm | Nhóm phải xuất hiện | Kết quả |
|---|---|---|
| Đăng nhập bằng tài khoản thường | i Truy cập | Thành công |
| Đăng nhập sai mật khẩu 1 lần | i Truy cập | **Thất bại** |
| Sai đủ số lần cho tới khi bị khóa | i + **iv Quản lý tài khoản** | Hai dòng khác nhóm |
| Gõ một tên đăng nhập không tồn tại | i Truy cập | Thất bại |
| Đăng nhập bằng tài khoản **admin** | **ii Đăng nhập quản trị** | Thành công |
| Đăng xuất | i Truy cập | Thành công |
| Đổi một ô cấu hình rồi bấm Cập nhật | **v Thay đổi cấu hình** | Chi tiết có **TRƯỚC/SAU** |
| Thêm một quy tắc địa chỉ | v Thay đổi cấu hình | Thành công |
| Truy cập `/admin` từ IP bị cấm | ii Đăng nhập quản trị | Thất bại |
| Gọi một URL gây lỗi | **iii Lỗi phát sinh** | Thất bại, chi tiết có vết gọi |

Kiểm luôn hai điều dễ sai:

- **Múi giờ**: cột Thời gian phải khớp với đồng hồ máy bạn, không lệch 7 tiếng.
- **Phân trang**: đổi bộ lọc khi đang ở trang 5 phải nhảy về trang 1, không
  hiện màn hình trống.

Và kiểm tra rằng file `App_Data\gPortal.txt` **không** có dòng nào chứa
`NHATKY_ROI` — nếu có thì nghĩa là có sự kiện đã không xuống được CSDL.
