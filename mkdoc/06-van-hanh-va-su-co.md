# 06 — Vận hành và xử lý sự cố

---

## 1. Triển khai: thứ tự bắt buộc

```
1.  Chạy Database/DbUpdate.sql          ← TRƯỚC
2.  Triển khai mã nguồn (gPortal.dll)
3.  Build + triển khai gPortalAdmin
4.  Cấu hình chính sách trong màn hình quản trị
```

### Vì sao migration phải chạy TRƯỚC

Mô hình EF trong mã nguồn mới khai báo 5 cột và 1 bảng mà CSDL cũ chưa có. Khi
EF sinh câu `SELECT`, nó liệt kê **tất cả** các cột đã khai báo. Thiếu một cột
→ SQL Server báo lỗi → `CacheHelper.GetCurrentPortalSettings()` ném ngoại lệ →
**toàn bộ portal sập**, không riêng gì tính năng mới.

Triển khai mã nguồn trước migration là một khoảng thời gian portal chết hoàn
toàn. Chạy migration trước thì hoàn toàn an toàn: mã nguồn cũ đơn giản là không
biết tới các cột mới.

---

## 2. Cách viết migration trong dự án này

`Database/DbUpdate.sql` là **một tệp duy nhất chạy được nhiều lần**. Mỗi thay
đổi là một khối `[MIGRATION 00X]` có mã số tăng dần.

Khuôn mẫu:

```sql
IF COL_LENGTH(N'dbo.<bảng>', N'<cột>') IS NULL
BEGIN
    ALTER TABLE dbo.<bảng> ADD <cột> <kiểu> NULL;
    PRINT N'[00X] Da them cot ...';
END
ELSE
    PRINT N'[00X] Cot ... da ton tai - bo qua.';
GO

IF NOT EXISTS (SELECT 1 FROM dbo.gp_DbMigrations WHERE MigrationId = '00X_<mã>')
    INSERT INTO dbo.gp_DbMigrations (MigrationId, Description) VALUES ('00X_<mã>', N'...');
GO
```

`gp_DbMigrations` chỉ để **ghi nhật ký** (biết khối nào đã chạy, chạy khi nào).
Bản thân các khối tự bảo vệ bằng `COL_LENGTH` / `OBJECT_ID`, nên chạy lại cả
tệp bao nhiêu lần cũng không sao.

### Danh sách migration

| Mã | Nội dung | Có backfill |
|---|---|---|
| 000 | Tạo bảng `gp_DbMigrations` | — |
| 001 | `gp_Users.MustChangePassword` | ✅ `DEFAULT 0` |
| 002 | Sửa chính sách mật khẩu trong `gp_PortalSettings` | — |
| 003 | `PasswordChangedUtc` + hạn sử dụng mật khẩu | ✅ Có |
| 004 | `FailedAttemptWindowMinutes`, `LastFailedAttemptUtc` | ❌ **Cố ý không** |
| 005 | `SessionTimeoutMinutes` | — |
| 006 | `SessionWarnBeforeMinutes` | — |
| 007 | Bảng `gp_AdminIpRules` | ❌ **Cố ý không** |

### Vì sao 003 phải backfill mà 004, 007 thì không

Không phải sở thích — nó phụ thuộc vào **NULL có nghĩa gì** trong từng trường
hợp:

| Migration | `NULL` nghĩa là | Hậu quả nếu để NULL |
|---|---|---|
| 003 `PasswordChangedUtc` | "Không biết đổi mật khẩu lần cuối khi nào" | Có nguy cơ bị hiểu thành "đã hết hạn từ lâu" → **khóa nhầm toàn bộ người đang dùng** |
| 004 `LastFailedAttemptUtc` | "Không biết lần sai trước khi nào" | Vô hại — code kiểm `HasValue`, không thỏa thì giữ nguyên hành vi cũ |
| 007 bảng rỗng | "Chính sách chưa được thiết lập" | Vô hại — rỗng = tắt tính năng |

Ở 004, backfill bằng `GETUTCDATE()` sẽ **sai**: nó tuyên bố mọi tài khoản vừa
mới nhập sai mật khẩu tại thời điểm chạy migration.

Ở 007, chèn dữ liệu mẫu sẽ **khóa toàn bộ quản trị viên** ngay lập tức nếu địa
chỉ mẫu không đúng.

> **Câu hỏi cần đặt cho mỗi cột mới:** "Nếu để NULL, code sẽ hiểu nó thành gì,
> và hậu quả là gì?" Chỉ backfill khi câu trả lời là có hại.

---

## 3. Sự cố: bị khóa khỏi phần quản trị

**Triệu chứng:** mọi trang `/admin/...` trả về 403 kèm thông báo địa chỉ mạng.

### Cách sửa nhanh

Trên máy chủ, mở `gPortal/Web.config`, thêm vào `<appSettings>`:

```xml
<add key="gp:AdminIpRestrictionDisabled" value="true" />
```

Lưu file → IIS tự khởi động lại ứng dụng → chốt chặn tắt hoàn toàn.

Vào lại, sửa chính sách cho đúng, rồi **xóa (hoặc chú thích) dòng đó đi**.

### Nếu không truy cập được máy chủ

Chạy SQL trực tiếp:

```sql
-- Xem chính sách hiện tại
SELECT Id, IpValue, Description, Enabled, CreatedBy FROM dbo.gp_AdminIpRules;

-- Tắt toàn bộ (rỗng/tắt hết = tính năng TẮT)
UPDATE dbo.gp_AdminIpRules SET Enabled = 0;

-- Hoặc thêm địa chỉ của mình
INSERT INTO dbo.gp_AdminIpRules (Id, IpValue, Description, Enabled, CreatedUtc, CreatedBy)
VALUES (NEWID(), '203.0.113.7', N'Bo sung khan cap', 1, GETUTCDATE(), 'dba');
```

Có hiệu lực **chậm nhất sau 60 giây** (thời gian giữ bộ nhớ tạm).

---

## 4. Sự cố: mọi người dùng bị khóa vì mật khẩu

**Triệu chứng:** ai đăng nhập cũng bị đẩy tới trang đổi mật khẩu.

Nguyên nhân hay gặp: `PasswordValidityDays` bị đặt quá nhỏ, hoặc
`PasswordChangedUtc` của dữ liệu cũ không được backfill đúng.

```sql
-- Xem cấu hình
SELECT PasswordChangeIntervalDays, PasswordValidityDays FROM dbo.gp_PortalSettings;

-- Tắt tính năng: cả HAI cùng NULL (cùng có hoặc cùng trống)
UPDATE dbo.gp_PortalSettings
SET PasswordChangeIntervalDays = NULL, PasswordValidityDays = NULL;

-- Xem có ai chưa có mốc thời gian không
SELECT COUNT(*) FROM dbo.gp_Users WHERE PasswordChangedUtc IS NULL;
```

Có hiệu lực **ngay lập tức** — cấu hình được đọc lại ở mỗi request, không có bộ
nhớ tạm nào giữ nó.

---

## 5. Sự cố: người dùng bị đăng xuất liên tục

```sql
SELECT SessionTimeoutMinutes, SessionWarnBeforeMinutes FROM dbo.gp_PortalSettings;

-- Tắt
UPDATE dbo.gp_PortalSettings SET SessionTimeoutMinutes = NULL, SessionWarnBeforeMinutes = NULL;
```

Nhớ: `SessionTimeoutMinutes` có **trần 1440 phút (1 ngày)**, bằng
`ExpireTimeSpan` của cookie xác thực trong `Startup.Auth.cs`. Đặt lớn hơn thì
cookie hết hạn trước và con số cấu hình trở thành lời hứa suông. Trần này được
ép ở `AdminController`.

---

## 6. Sự cố: tài khoản bị khóa oan

```sql
-- Xem
SELECT UserName, AccessFailedCount, LastFailedAttemptUtc, LockoutEndDateUtc
FROM dbo.gp_Users WHERE UserName = '<tên>';

-- Mở khóa một tài khoản
UPDATE dbo.gp_Users
SET AccessFailedCount = 0, LockoutEndDateUtc = NULL, LastFailedAttemptUtc = NULL
WHERE UserName = '<tên>';
```

Nếu nhiều người bị khóa cùng lúc, kiểm tra `MaxInvalidPasswordAttempts` có bị
đặt thành `0` không — giá trị đó khóa ngay ở lần nhập sai **đầu tiên**. Xem
[02-khoa-tai-khoan.md](02-khoa-tai-khoan.md) mục 4.

---

## 7. Nơi xem log

Log dùng **log4net**, cấu hình ở `log4net.config`.

Các chốt chặn ghi log theo quy ước:

| Mức | Khi nào | Ví dụ |
|---|---|---|
| `Warn` | Có người bị chặn (bình thường, đúng thiết kế) | `AdminIpGuard: CHAN truy cap quan tri tu ... toi ...` |
| `Error` | Chốt chặn **không làm việc được** | `AdminIpGuard: khong doc duoc quy tac. ...` |

`Error` từ một chốt chặn nghĩa là **chốt đó đang không bảo vệ gì cả** (hoặc
đang chạy bằng dữ liệu cũ trong bộ nhớ). Đây là loại lỗi cần xử lý ngay, dù
người dùng không thấy gì bất thường.

Tra nhanh:

```
AdminIpGuard
SessionTimeoutGuard
MustChangePasswordGuard
```

---

## 8. Cấu hình hạ tầng trong `Web.config`

| Khóa | Mặc định | Khi nào cần đổi |
|---|---|---|
| `gp:AdminIpRestrictionDisabled` | không có (= bật chốt) | **Chỉ khi** đã lỡ tự khóa |
| `gp:AdminIpTrustedProxies` | không có (= không tin `X-Forwarded-For`) | **Chỉ khi** portal đứng sau proxy / cân bằng tải |

### Vì sao hai khóa này ở `Web.config` chứ không ở giao diện

Chúng là **cấu hình hạ tầng** — đặt một lần lúc triển khai, không phải chính
sách để quản trị viên chỉnh hằng ngày.

Quan trọng hơn: nếu để trong giao diện, **một lần cấu hình sai có thể khóa
chính cái giao diện dùng để sửa nó**. Đặt ở `Web.config` thì đường lùi nằm
ngoài vòng lặp đó.

### Cảnh báo về `gp:AdminIpTrustedProxies`

- **Để trống nếu portal nối thẳng ra Internet/LAN.** Khi trống, hệ thống chỉ
  tin địa chỉ TCP thật và **không đọc** `X-Forwarded-For` — đúng như vậy, vì
  tiêu đề đó do client tự đặt.
- **Chỉ điền khi thật sự có proxy.** Điền nhầm một dải rộng nghĩa là cho phép
  bất kỳ ai trong dải đó **tự khai địa chỉ của mình** và vượt chốt.

---

## 9. Khi thêm chức năng quản trị mới

Nếu bạn thêm một controller hoặc trang quản trị mới **nằm ngoài** `/admin` và
`/gclient/gadmin`, hãy nhớ:

- Vế (2) của `AdminIpGuard` (soi theo **vai trò người dùng**) vẫn phủ được nó,
  miễn là người dùng có vai trò `Admins`.
- Nhưng vế (1) (soi theo **đường dẫn**, áp dụng cả với khách chưa đăng nhập) sẽ
  không phủ. Nếu trang mới cần chặn từ trước khi xác thực, hãy bổ sung đường
  dẫn vào `AdminIpGuard.DuongDanQuanTri`.

Tương tự với `MustChangePasswordGuard.AllowedPaths` nếu trang mới cần được truy
cập trong lúc người dùng đang bị buộc đổi mật khẩu.
