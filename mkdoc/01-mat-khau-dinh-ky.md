# 01 — Chính sách mật khẩu định kỳ

## Yêu cầu nghiệp vụ

| Mã | Nội dung |
|---|---|
| (a) | Buộc đổi mật khẩu khi tài khoản đang dùng mật khẩu do quản trị cấp |
| (c) | Có thiết lập thời gian **yêu cầu** người dùng đổi mật khẩu |
| (d) | Có thiết lập thời gian mật khẩu còn **hợp lệ** |
| (đ) | Quá hạn hợp lệ thì **khóa** tài khoản |
| (e) | Đổi mật khẩu thành công thì **mở khóa** |

---

## 1. (a) Mật khẩu mặc định — buộc đổi ở lần đăng nhập đầu

Đây là yêu cầu **đứng đầu vòng đời** của một tài khoản: quản trị tạo tài khoản
và cấp một mật khẩu ban đầu, người dùng phải tự đặt lại trước khi làm được bất
cứ việc gì.

Toàn bộ tính năng gói trong **một cột `BIT`**: `gp_Users.MustChangePassword`.
Cái đáng học không nằm ở cột đó, mà ở **ai bật nó, ai tắt nó, và bật/tắt sai
thì hỏng ra sao**.

### 1.1. Hai chữ "mặc định" ở hai thời điểm khác nhau

Chỗ này rất dễ nhầm, và nhầm thì hậu quả tức thì trên toàn bộ người dùng.

Migration 001 tạo cột với **`DEFAULT (0)`** — tức là *không* bắt buộc đổi:

```sql
ALTER TABLE dbo.gp_Users
    ADD MustChangePassword BIT NOT NULL
        CONSTRAINT DF_gp_Users_MustChangePassword DEFAULT (0);
```

Nghe có vẻ ngược với yêu cầu — "mặc định" lẽ ra phải là *bật* chứ? Không:

> `ALTER TABLE ... ADD ... NOT NULL DEFAULT` sẽ **điền giá trị đó vào toàn bộ
> các dòng đang có**. Để `DEFAULT (1)` thì ngay giây chạy script, **mọi người
> dùng hiện hữu trong công ty** đều bị buộc đổi mật khẩu — kể cả người vừa tự
> đổi hôm qua.

Nên "mặc định bật" chỉ áp dụng cho **người dùng MỚI**, và được xử lý ở **tầng
ứng dụng**, không phải tầng CSDL:

| Tầng | Giá trị | Áp dụng cho |
|---|---|---|
| CSDL (migration 001) | `DEFAULT 0` | Người dùng **đã có** khi chạy script |
| Ứng dụng (`OrganizationController`) | `?? true` | Người dùng **tạo mới** từ nay |
| Giao diện (`mUser.js`) | `defaultValue: true` | Ô tick sẵn khi bấm "Thêm" |

Ba dòng trên trông như trùng lặp. Chúng không trùng: chúng nói về **ba nhóm
người khác nhau tại ba thời điểm khác nhau**.

### 1.2. Cờ được BẬT ở hai chỗ, không phải một

Chỗ thứ nhất thì ai cũng nghĩ ra — lúc tạo tài khoản:

```csharp
// OrganizationController.CreateUserInOrg
MustChangePassword = model.MustChangePassword ?? true,
PasswordChangedUtc = DateTime.UtcNow
```

Chỗ thứ hai là chỗ hay bị quên — lúc quản trị **cấp lại** mật khẩu:

```csharp
// OrganizationController.ResetPasswordUser
bool mustChange = mustChangePassword ?? true;
```

Vì sao phải bật lại? Vì bản chất của yêu cầu (a) **không phải "lần đăng nhập
đầu tiên"**, mà là *"mật khẩu này hiện đang là một bí mật mà quản trị viên
biết"*. Quản trị cấp lại mật khẩu thì tình trạng đó **tái lập y hệt** lúc mới
tạo tài khoản. Chỉ bật cờ lúc tạo mà quên lúc reset thì có một lối đi vòng:
người dùng đổi mật khẩu → gọi hỗ trợ xin cấp lại → từ đó dùng mãi mật khẩu do
quản trị đặt.

> **Nguyên tắc:** đặt cờ theo **điều kiện thực tế nó mô tả**, không theo *sự
> kiện* mà bạn tình cờ nghĩ tới đầu tiên. "Lần đăng nhập đầu" là một sự kiện;
> "mật khẩu do người khác biết" mới là điều kiện.

### 1.3. Quy ước `null = true`

Cả hai chỗ đều dùng `?? true`, không phải `?? false`. Đây là lựa chọn có chủ ý:

- Ứng dụng di động và các client cũ **chưa gửi trường này** → nhận về `null`.
- `?? true` nghĩa là **thiếu thông tin thì chọn hướng an toàn**: cứ bắt đổi.
- Muốn tắt thì client phải nói **tường minh** `false`.

Đây chính là nguyên tắc 3 trong [README](README.md): *chọn hướng hỏng theo hậu
quả*. Nếu để `?? false`, một client cũ quên gửi trường sẽ **âm thầm tạo ra tài
khoản không bao giờ bị buộc đổi mật khẩu** — mà không có thông báo lỗi nào.

### 1.4. Cờ được TẮT ở đúng một chỗ

```csharp
// ManageController.ChangePassword - sau khi đổi thành công
user.MustChangePassword = false;
user.PasswordChangedUtc = DateTime.UtcNow;
await UserManager.UpdateAsync(user);
await SignInManager.SignInAsync(user, isPersistent: false, rememberBrowser: false);
```

Hai dòng, hai tác dụng khác nhau — và **một dòng giải quyết luôn ba yêu cầu**:

| Dòng | Giải quyết |
|---|---|
| `MustChangePassword = false` | (a) thoát trạng thái mật khẩu mặc định |
| `PasswordChangedUtc = UtcNow` | (c) tuổi mật khẩu về 0 **và** (e) **mở khóa** tài khoản đã hết hiệu lực |

Yêu cầu (e) "đổi mật khẩu thì mở khóa" **không có dòng code nào riêng** — xem
mục 3 dưới đây để hiểu vì sao.

Phải cập nhật **trước khi rời trang**: chốt chặn đọc từ CSDL ở mỗi request, nên
nếu chuyển hướng trước rồi mới ghi, request kế tiếp vẫn thấy cờ cũ và đá người
dùng ngược về trang đổi mật khẩu.

### 1.5. Một cái bẫy có thật trong `ResetPasswordUser`

```csharp
var result = await UserManager.ResetPasswordAsync(userId, resetToken, newPass);
...
// Đọc LẠI đối tượng, không dùng lại biến `user` ở trên
var userAfterReset = await UserManager.FindByIdAsync(userId);
userAfterReset.MustChangePassword = mustChange;
```

`ResetPasswordAsync` **đã ghi xuống CSDL** rồi (sinh `PasswordHash` mới và
`SecurityStamp` mới). Đối tượng `user` nạp trước đó giờ đã **cũ**. Gán cờ lên
nó rồi `UpdateAsync` sẽ ghi đè **toàn bộ hàng** bằng dữ liệu cũ → **mật khẩu
vừa đặt bị xóa mất**, người dùng nhận mật khẩu mới nhưng đăng nhập không được.

Đây là lớp lỗi khó tìm nhất: không có ngoại lệ, không có log, mọi lời gọi đều
báo thành công.

### 1.6. Vì sao (a) chặn cửa ngay, còn (c) chỉ nhắc

```csharp
public static bool IsBlocking(PasswordChangeReason reason)
{
    return reason == PasswordChangeReason.Expired
        || reason == PasswordChangeReason.DefaultPassword;   // (a) chặn
}
```

Khác nhau ở chỗ **ai đang biết mật khẩu**:

- **(c) tới hạn định kỳ:** mật khẩu đó do **chính người dùng** đặt, chưa có dấu
  hiệu lộ. Chặn ngay là làm phiền vô cớ → cho ân hạn kèm đếm ngược.
- **(a) mật khẩu mặc định:** ít nhất **hai người** biết mật khẩu này, và người
  dùng **chưa từng tự chọn gì cả**. Để càng lâu càng rủi ro, mà cái giá của
  việc chặn lại rất thấp — họ chưa có việc gì đang làm dở.

Trong `Evaluate`, `Expired` được kiểm **trước** `DefaultPassword`. Cả hai đều
chặn nên thứ tự không đổi hành vi, chỉ đổi **câu thông báo** hiện ra — và câu
"mật khẩu đã hết hiệu lực" là câu đúng hơn khi cả hai điều kiện cùng đúng.

### 1.7. Quản trị viên nhìn thấy gì

| Nơi | Thành phần |
|---|---|
| Form tạo người dùng | Checkbox *"Yêu cầu người dùng đặt mật khẩu mới ở lần đăng nhập đầu tiên"* (`chbDoiMatKhauLanDau`) |
| Form thiết lập lại mật khẩu | Checkbox tương tự (`chkDoiMatKhau`) |
| Lưới danh sách người dùng | Cột **"Chờ đổi MK"** — hiện ⚠️ với tài khoản còn dùng mật khẩu quản trị cấp |

Checkbox trên form người dùng **bị ẩn khi sửa** (`hidden: '{record.Id != ""}'`),
cùng lý do với ô Mật khẩu: mật khẩu mặc định chỉ tồn tại **ở thời điểm cấp**.
Muốn bật lại cờ cho tài khoản đã có thì phải đi qua *"Thiết lập lại mật khẩu"*
— tức là phải **thực sự cấp một mật khẩu mới**, chứ không thể chỉ tick một ô.

Nếu cho tick lại trên form sửa, ta sẽ tạo ra được trạng thái vô nghĩa: *"tài
khoản đang dùng mật khẩu mặc định"* trong khi **không ai cấp mật khẩu mặc định
nào cả** — người dùng bị chặn cửa và không ai biết mật khẩu để đưa cho họ.

Cột ⚠️ trong lưới là thứ trả lời câu hỏi vận hành: *"phát tài khoản tuần trước,
ai chưa vào lần nào?"*

---

## 2. Ba trạng thái, không phải hai

Đây là phần cốt lõi và cũng là chỗ bản đầu tiên làm sai.

```
   Ngày đổi mật khẩu                                  Hôm nay
        │                                                │
        ├────────────────────┬────────────────┬──────────┤
        │   BÌNH THƯỜNG      │   ÂN HẠN       │  KHÓA    │
        0                   60 ngày         90 ngày
                    PasswordChangeIntervalDays   PasswordValidityDays
                              (c)                     (d)
```

| Vùng | Người dùng thấy gì | Có dùng được hệ thống không |
|---|---|---|
| Bình thường | Không có gì | ✅ Có |
| **Ân hạn** | Nhắc kèm **đếm ngược số ngày** + nút "Để sau" | ✅ **Có** |
| Khóa | Chỉ có trang đổi mật khẩu | ❌ Không |

### Vì sao phải có vùng ân hạn

Bản đầu tiên chặn ngay tại mốc (c). Hậu quả:

1. Mốc (d) **không bao giờ tới lượt** — người dùng đã bị khóa từ ngày 60, nên
   ngày 90 không có ý nghĩa gì. Hệ thống có hai ô cấu hình mà chỉ một ô hoạt
   động.
2. Về nghiệp vụ, chính tên gọi đã nói rõ: (c) là "thời gian **yêu cầu** đổi",
   (d) là "thời gian còn **hợp lệ**". Yêu cầu là lời nhắc, hết hợp lệ mới là
   khóa. Chặn ở (c) tức là hiểu "yêu cầu" thành "cấm".

### Cách sửa: tách sự thật khỏi chính sách

```csharp
// "VÌ SAO nên đổi mật khẩu" — một sự thật về người dùng
public static PasswordChangeReason Evaluate(ApplicationUser user, gp_PortalSettings settings)

// "Lý do đó có CẤM truy cập không" — một quyết định chính sách
public static bool IsBlocking(PasswordChangeReason reason)
{
    return reason == PasswordChangeReason.Expired
        || reason == PasswordChangeReason.DefaultPassword;
}
```

`ChangeIntervalReached` **không** nằm trong danh sách chặn. Đó là toàn bộ thay
đổi về mặt hành vi — nhưng nó chỉ đúng vì hai câu hỏi đã được tách ra; gộp lại
thì không có chỗ nào để diễn đạt "có lý do nhưng không chặn".

---

## 3. Trạng thái "khóa" KHÔNG được lưu ở đâu cả

Đây là quyết định thiết kế đáng nói nhất trong tính năng này.

Cách làm thông thường sẽ là thêm một cột `IsLocked BIT`, rồi viết một job nền
quét định kỳ để bật cột đó lên, và một đoạn code khác để tắt nó đi khi người
dùng đổi mật khẩu.

**Ở đây không có cột nào cả.** Trạng thái khóa được **tính ra** mỗi lần kiểm
tra, từ `PasswordChangedUtc`:

```csharp
double tuoiNgay = (DateTime.UtcNow - user.PasswordChangedUtc.Value).TotalDays;
if (tuoiNgay >= settings.PasswordValidityDays.Value)
    return PasswordChangeReason.Expired;
```

### Ba thứ được miễn phí nhờ lựa chọn đó

1. **Yêu cầu (e) không cần code riêng.** Đổi mật khẩu ghi `PasswordChangedUtc`
   = hiện tại → phép tính cho ra "chưa hết hạn" → khóa tự tan. Không có bước
   "nhớ mở khóa" nào để quên.
2. **Không cần job nền.** Chốt chặn đã chạy ở mỗi request rồi.
3. **Sửa cấu hình có hiệu lực ngay lập tức.** Quản trị viên đổi từ 90 xuống 60
   ngày là phép tính đổi theo ngay ở request kế tiếp — không phải chờ job chạy.

> **Nguyên tắc rút ra:** trạng thái **suy ra được** thì không thể lệch với thực
> tế. Trạng thái **lưu trữ** thì luôn có nguy cơ ai đó quên cập nhật một nhánh.

---

## 4. Vì sao đọc từ CSDL chứ không đọc từ cookie

```csharp
/// Đọc cờ và mốc thời gian từ CSDL, KHÔNG đọc từ claim trong cookie:
/// nội dung cookie bị đóng băng lúc đăng nhập, nên sau khi người dùng
/// đổi mật khẩu xong, cookie cũ vẫn mang giá trị cũ và họ bị đá về trang
/// đổi mật khẩu mãi mãi.
```

Đọc claim thì nhanh hơn nhiều (không tốn câu SELECT nào). Nhưng claim được ghi
vào vé **lúc đăng nhập** và không tự cập nhật. Người dùng đổi mật khẩu xong,
claim vẫn ghi "phải đổi mật khẩu" → bị đẩy về trang đó → đổi tiếp → vẫn bị
đẩy → vòng lặp không lối ra.

Giá phải trả: một câu `SELECT` cho mỗi request. Được giảm bằng hai cách:

- Kết quả được nhớ trong `HttpContext.Items` → mỗi request tốn đúng một câu dù
  bị hỏi nhiều lần.
- Các đường dẫn tĩnh (`.js`, `.css`, ảnh…) được bỏ qua ngay từ đầu.

---

## 5. Một lỗ hổng có thật đã được vá

Danh sách đường dẫn cho qua ban đầu được so bằng `Contains`:

```csharp
// BẢN CŨ - CÓ LỖ HỔNG
return AllowedPaths.Any(p => lower.Contains(p));
```

Nghĩa là **bất kỳ URL nào CHỨA chuỗi đó ở bất kỳ vị trí nào** đều thoát chốt:

```
/account/login                    →  cho qua  (đúng)
/api/report/account/login-history →  cho qua  (SAI - lọt chốt)
```

### Nhưng vì sao bản cũ lại chọn `Contains`?

Đây là câu hỏi quan trọng, vì đổi thẳng sang `StartsWith` sẽ tạo ra một sự cố
khác. Nếu portal chạy trong **thư mục ảo**, đường dẫn thô luôn kèm tiền tố:

```
/portal/Manage/ChangePassword
```

`StartsWith("/manage/changepassword")` → **trượt**. Kết quả: chính trang đổi
mật khẩu cũng bị chặn → vòng chuyển hướng vô tận → portal không dùng được.

`Contains` chịu được thư mục ảo. Đó gần như chắc chắn là lý do nó tồn tại.

### Bản sửa làm cả hai việc

```csharp
// 1. Quy về đường dẫn tương đối với ứng dụng TRƯỚC
//    "/portal/Manage/ChangePassword" → "/manage/changepassword"
relative = VirtualPathUtility.ToAppRelative(path).TrimStart('~').TrimEnd('/').ToLowerInvariant();

// 2. Rồi mới so theo TRỌN ĐOẠN
return danhSach.Any(p =>
    relative.Equals(p, StringComparison.Ordinal) ||
    relative.StartsWith(p + "/", StringComparison.Ordinal));
```

Logic này nay nằm ở `Authorization/DuongDanRequest.cs` và được **cả hai chốt**
(`MustChangePasswordGuard`, `AdminIpGuard`) dùng chung — hai bản viết tay sẽ có
ngày lệch nhau, và lệch ở chốt chặn nghĩa là một bên có lỗ mà bên kia không có.

> **Bài học:** khi thấy một đoạn code "sai rành rành", hãy tìm cho ra lý do nó
> được viết như vậy trước đã. Sửa mà không biết lý do cũ thì rất dễ đổi một lỗ
> hổng lấy một sự cố.

---

## 6. Ràng buộc cấu hình: cùng có hoặc cùng trống

`AdminController.KiemTraCauHinhHanMatKhau` từ chối lưu nếu chỉ điền một trong
hai ô:

| Trường hợp | Hậu quả nếu cho phép |
|---|---|
| Chỉ có (c), thiếu (d) | Người dùng bị nhắc nhưng **không có hạn chót nào** → chính sách chỉ còn là lời khuyên |
| Chỉ có (d), thiếu (c) | Tài khoản bị khóa đột ngột mà **chưa hề được nhắc lấy một lần** — tệ nhất |
| (d) < (c) | Bị khóa **trước cả khi** kịp được nhắc |

Chặn ngay ở tầng lưu thì ba tình huống trên không tồn tại, và luật trong
`MustChangePasswordGuard` chỉ còn phải lo đúng **một** hình dạng dữ liệu.

Ràng buộc này được viết ở **cả hai nơi**:

- `PortalWebsite.js` → hàm `PortalWebsiteKiemTraHanMatKhau` (tô đỏ ô nhập)
- `AdminController.cs` → `KiemTraCauHinhHanMatKhau` (chốt thật)

Trong ExtJS có một chi tiết dễ quên: ràng buộc nằm **giữa hai ô**, nên sửa ô
này có thể làm ô kia từ sai thành đúng. ExtJS **không** tự chạy lại validator
của ô kia, nên phải gọi tay trong `listeners.change` — nếu không, vệt đỏ sẽ
đứng lại ở trạng thái cũ và nói dối người dùng.

---

## 7. Nơi cần biết

| Việc | Tệp |
|---|---|
| Luật quyết định (`Evaluate` / `IsBlocking`) | `gPortal/Authorization/MustChangePasswordGuard.cs` |
| Chốt chặn ở mọi request | `Global.asax.cs` → `Application_PostAuthenticateRequest` |
| Điều hướng sau đăng nhập | `AccountController.RedirectAfterSignIn` |
| Màn hình đổi mật khẩu | `Views/Manage/ChangePassword.cshtml` |
| **BẬT** cờ (a) khi tạo tài khoản | `OrganizationController.CreateUserInOrg` |
| **BẬT** cờ (a) khi cấp lại mật khẩu | `OrganizationController.ResetPasswordUser` |
| **TẮT** cờ (a) khi người dùng tự đổi | `ManageController.ChangePassword` |
| Checkbox lúc tạo người dùng | `gOrgAdmin/app/view/Users/FrmOrganizationUser.js` |
| Checkbox lúc cấp lại mật khẩu | `gOrgAdmin/app/view/Users/FrmResetPassUser.js` |
| Cột ⚠️ "Chờ đổi MK" | `gOrgAdmin/app/view/Users/OrganizationUsers.js` |
| Giá trị mặc định phía giao diện | `gOrgAdmin/app/model/mUser.js` |
| Kiểm tra cấu hình | `AdminController.KiemTraCauHinhHanMatKhau` |
| Giao diện cấu hình | `PortalWebsite.js` — tab **"Mật khẩu"** |
| Migration | `Database/DbUpdate.sql` — 001 (cờ a), 002, 003 |

---

## 8. Việc còn treo

- **Ứng dụng di động** đang đọc cờ `MustChangePassword`. Ý nghĩa của cờ đó đã
  bị thu hẹp (giờ chỉ còn là "(a) mật khẩu mặc định"), nên app cần được cập
  nhật để hiểu thêm trạng thái ân hạn.
- Nên kiểm tra dữ liệu cũ xem có bản ghi nào chỉ điền một trong hai ô không:
  ```sql
  SELECT Id, PasswordChangeIntervalDays, PasswordValidityDays
  FROM dbo.gp_PortalSettings;
  ```
  Ràng buộc mới chỉ chặn từ lúc lưu trở đi, không dọn dữ liệu đã có.
