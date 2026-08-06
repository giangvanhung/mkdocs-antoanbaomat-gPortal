# 00 — Tổng quan kiến trúc

Đọc file này trước. Nó giải thích **khung sườn chung** mà cả bốn tính năng đều
dùng lại; các file sau chỉ nói phần riêng của từng tính năng.

---

## 1. Vấn đề nền: portal này là hệ LAI

```
                     ┌─────────────────────────────┐
   Trình duyệt  ───► │   IIS  /  ASP.NET pipeline  │
                     └──────────────┬──────────────┘
                                    │
                   ┌────────────────┴────────────────┐
                   ▼                                 ▼
        ┌──────────────────┐              ┌────────────────────┐
        │  MVC Controller  │              │  WebForms (.aspx)  │
        │  AccountController│              │  24 trang          │
        │  AdminController │              │  admin/portal.aspx │
        │  ManageController│              │  apps/home.aspx    │
        └──────────────────┘              └────────────────────┘
                   │                                 │
        ĐI QUA GlobalFilters               KHÔNG đi qua GlobalFilters
```

Đây là sự thật quan trọng nhất trong toàn bộ thiết kế.

Nếu đặt rào chắn bằng `ActionFilter` của MVC (cách mà mọi hướng dẫn trên mạng
đều dạy), thì nó **chỉ bảo vệ nửa trái của sơ đồ**. Nửa phải — trong đó có
`admin/portal.aspx`, tức là chính màn hình quản trị — hoàn toàn không bị chặn.
Gõ thẳng URL là vào.

### Lời giải: đặt chốt ở tầng HttpApplication

`Global.asax.cs` → `Application_PostAuthenticateRequest`:

```csharp
protected void Application_PostAuthenticateRequest(object sender, EventArgs e)
{
    if (SessionTimeoutGuard.Handle(HttpContext.Current)) return;
    if (AdminIpGuard.Handle(HttpContext.Current)) return;
    MustChangePasswordGuard.Handle(HttpContext.Current);
}
```

Đây là điểm hội tụ: **mọi** request — MVC, WebForms, API, ảnh, file css — đều
đi qua đây.

### Vì sao là `PostAuthenticateRequest`, không phải `BeginRequest`?

| Giai đoạn | `HttpContext.User` | `Session` |
|---|---|---|
| `BeginRequest` | ❌ chưa có | ❌ chưa có |
| **`PostAuthenticateRequest`** | ✅ **đã có** | ❌ chưa có |
| `AcquireRequestState` | ✅ có | ✅ có |
| `PreRequestHandlerExecute` | ✅ có | ✅ có |

Chốt chặn cần biết **ai đang đăng nhập** → không thể ở `BeginRequest`.

Và vì `Session` chưa sẵn sàng ở giai đoạn này, **không chốt nào được phép dùng
`Session`**. Đó là lý do mốc thời gian hoạt động của phiên được cất trong **vé
xác thực** chứ không phải trong `Session` — xem [03](03-thoi-gian-cho-phien.md).

---

## 2. Thứ tự ba chốt chặn — và vì sao thứ tự đó không tùy tiện

```
   1. SessionTimeoutGuard      →  "Phiên này còn sống không?"
   2. AdminIpGuard             →  "Người này có ngồi ở nơi được phép không?"
   3. MustChangePasswordGuard  →  "Mật khẩu này còn dùng được không?"
```

Mỗi lần đảo thứ tự đều tạo ra một hành vi sai:

**Nếu đảo 1 và 3:** người có phiên đã hết hạn sẽ bị đẩy tới trang đổi mật khẩu
rồi *mới* bị đá ra. Hai lần chuyển hướng, và thông báo hiển thị sai ngữ cảnh —
họ đọc "hãy đổi mật khẩu" trong khi vấn đề thật là "bạn đã bị đăng xuất".

**Nếu đảo 2 và 3:** người ở địa chỉ mạng không hợp lệ sẽ được đẩy tới trang đổi
mật khẩu. Trang đó tồn tại nghĩa là **tài khoản có thật** — ta vừa xác nhận
điều đó cho một người mà chính sách đang muốn giữ ở ngoài.

Nguyên tắc chung: **chốt nào phủ định nhiều thứ hơn thì đứng trước.** "Bạn
không còn là ai cả" > "Bạn không được ở đây" > "Bạn cần làm việc X".

---

## 3. Khuôn mẫu chung của một chốt chặn

Cả ba chốt đều có cùng hình dạng, và điều đó là cố ý:

```csharp
public static bool Handle(HttpContext context)
{
    // 1. Có thuộc phạm vi áp dụng không?  (không → return false)
    // 2. Có vi phạm không?                (không → return false)
    // 3. Vi phạm:
    //      - request AJAX  → trả JSON kèm mã lỗi HTTP
    //      - request thường → chuyển hướng hoặc trả trang thông báo
    //      - CompleteRequest()
    //    return true;
}
```

### Vì sao AJAX phải xử lý khác?

Nếu trả `302 Redirect` cho một request AJAX, `XMLHttpRequest` **tự động đi
theo** chuyển hướng đó và nhận về... mã HTML của trang đăng nhập. Phía client
nhận một chuỗi HTML ở chỗ nó đang chờ JSON, và báo "dữ liệu hỏng" — một thông
báo hoàn toàn không liên quan tới nguyên nhân thật.

Vì vậy AJAX luôn được trả về mã lỗi thật (401 / 403) kèm JSON có `redirectUrl`,
để phía client tự quyết định.

---

## 4. Tách "sự thật" khỏi "chính sách"

Một khuôn mẫu lặp lại ở nhiều chỗ, đáng nhớ vì nó là cách sửa một lỗi thiết kế
có thật:

```csharp
Evaluate(user, settings)  →  VÌ SAO người này nên đổi mật khẩu    (sự thật)
IsBlocking(reason)        →  Lý do đó có CẤM truy cập không       (chính sách)
```

Bản đầu tiên gộp hai câu hỏi làm một: hễ có lý do là chặn. Hậu quả: tham số
"thời gian yêu cầu đổi mật khẩu" (60 ngày) và "thời gian mật khẩu hợp lệ"
(90 ngày) **hành xử y hệt nhau** — cái nào tới trước thì khóa. Nghĩa là tham số
thứ hai không bao giờ có tác dụng, và hệ thống có hai ô cấu hình khác tên nhưng
chỉ một ô có ý nghĩa.

Tách ra thì mỗi câu hỏi có đúng một nơi trả lời, và những nơi cần biết *lý do*
(để hiện thông báo) không phải suy diễn lại *luật chặn*.

---

## 5. Bản đồ tệp

### Chốt chặn — `gPortal/Authorization/`

| Tệp | Vai trò |
|---|---|
| `MustChangePasswordGuard.cs` | Chính sách mật khẩu |
| `SessionTimeoutGuard.cs` | Thời gian chờ của phiên |
| `AdminIpGuard.cs` | Giới hạn địa chỉ mạng quản trị |
| `KhopDiaChiMang.cs` | Luật đọc/so khớp địa chỉ IP (thuần tính toán) |
| `DuongDanRequest.cs` | Quy chuẩn đường dẫn, dùng chung cho các chốt |

### Điểm nối

| Tệp | Vai trò |
|---|---|
| `gPortal/Global.asax.cs` | Gọi cả ba chốt; nhúng script vào mọi trang `.aspx` |
| `gPortal/App_Start/Startup.Auth.cs` | Móc `OnValidateIdentity` cho đồng hồ phiên |
| `gPortal/Controllers/AccountController.cs` | Đăng nhập, đăng xuất, `SessionPolicy` |
| `gPortal/Controllers/AdminController.cs` | API cấu hình + kiểm tra hợp lệ phía máy chủ |

### Dữ liệu — `gPortal.Framework/Identity/IdentityModels.cs`

| Bảng / cột | Thuộc tính năng |
|---|---|
| `gp_Users.MustChangePassword` | 1 |
| `gp_Users.PasswordChangedUtc` | 1 |
| `gp_Users.LastFailedAttemptUtc` | 2 |
| `gp_PortalSettings.PasswordChangeIntervalDays` | 1 |
| `gp_PortalSettings.PasswordValidityDays` | 1 |
| `gp_PortalSettings.MaxInvalidPasswordAttempts` | 2 (đã có sẵn) |
| `gp_PortalSettings.LockoutMinutes` | 2 (đã có sẵn) |
| `gp_PortalSettings.FailedAttemptWindowMinutes` | 2 |
| `gp_PortalSettings.SessionTimeoutMinutes` | 3 |
| `gp_PortalSettings.SessionWarnBeforeMinutes` | 3 |
| `gp_AdminIpRules` (bảng mới) | 4 |

### Giao diện — `gPortal_gClient/gPortalAdmin/`

| Tệp | Vai trò |
|---|---|
| `app/model/mPortalSetting.js` | Khai báo trường; `allowNull` rất quan trọng — xem [02](02-khoa-tai-khoan.md) |
| `app/view/Website/PortalWebsite.js` | Cả ba tab cấu hình |

---

## 6. Một quy ước xuyên suốt: `NULL` = TẮT

Mọi tham số số học đều dùng kiểu `int?` (cho phép NULL), và **NULL luôn có
nghĩa là "không áp dụng tính năng này"**.

Đây không phải chi tiết vụn vặt. Nó là lý do tồn tại của dòng `allowNull: true`
trong `mPortalSetting.js`:

> Mặc định ExtJS ép `null` thành `0` với trường kiểu `int`. Thiếu `allowNull`
> thì quản trị viên **xóa trắng ô nhập** sẽ gửi về `0` chứ không phải `null`.
> Mà `MaxInvalidPasswordAttempts = 0` nghĩa là **khóa tài khoản ngay ở lần nhập
> sai đầu tiên**.

Một dòng cấu hình thiếu ở tầng giao diện biến "tắt tính năng" thành "siết chặt
nhất có thể". Đây là kiểu lỗi tệ nhất: nó không báo lỗi ở đâu cả, chỉ lặng lẽ
khóa tài khoản của người dùng.
