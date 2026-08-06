# 02 — Khóa tài khoản khi đăng nhập sai

## Yêu cầu nghiệp vụ

> (a) Có chức năng cho phép thiết lập **giới hạn số lần đăng nhập sai trong một
> khoảng thời gian**.

Ba chữ **"trong một khoảng thời gian"** là toàn bộ phần việc mới. Phần còn lại
ASP.NET Identity đã có sẵn.

---

## 1. Cái đã có sẵn và cái còn thiếu

ASP.NET Identity 2 có sẵn cơ chế khóa tài khoản, cấu hình ở
`gPortal.Framework/Identity/IdentityConfig.cs`:

| Tham số | Ý nghĩa | Nguồn |
|---|---|---|
| `MaxFailedAccessAttemptsBeforeLockout` | Sai bao nhiêu lần thì khóa | ✅ Identity có sẵn |
| `DefaultAccountLockoutTimeSpan` | Khóa trong bao lâu | ✅ Identity có sẵn |
| **Cửa sổ đếm** | Bao lâu không sai thì bộ đếm về 0 | ❌ **Identity KHÔNG có** |

### Vì sao thiếu cửa sổ đếm là một vấn đề thật

Identity lưu `AccessFailedCount` trong bảng `gp_Users` và **cộng dồn vô thời
hạn** cho tới khi đăng nhập đúng hoặc bị khóa.

Nghĩa là:

```
Ngày 01/01:  sai 4 lần   →  AccessFailedCount = 4
...ba tháng trôi qua, không ai đụng vào tài khoản...
Ngày 01/04:  sai 1 lần   →  AccessFailedCount = 5  →  KHÓA
```

Người dùng gõ nhầm **một lần duy nhất** sau ba tháng và bị khóa. Đó không phải
là "5 lần sai trong một khoảng thời gian" — đó là "5 lần sai trong cả đời tài
khoản".

---

## 2. Cách bổ sung: ghi đè `AccessFailedAsync`

Thêm một cột `gp_Users.LastFailedAttemptUtc`, và ghi đè hàm mà Identity gọi mỗi
lần có một lần đăng nhập sai:

```csharp
public override async Task<IdentityResult> AccessFailedAsync(string userId)
{
    var user = await FindByIdAsync(userId);
    if (user != null)
    {
        var settings = CacheHelper.GetCurrentPortalSettings();
        int? cuaSoPhut = settings == null ? null : settings.FailedAttemptWindowMinutes;

        // Lần sai gần nhất đã quá cũ → coi như bắt đầu lại từ đầu.
        if (cuaSoPhut.HasValue && cuaSoPhut.Value > 0
            && user.LastFailedAttemptUtc.HasValue
            && (DateTime.UtcNow - user.LastFailedAttemptUtc.Value).TotalMinutes > cuaSoPhut.Value)
        {
            user.AccessFailedCount = 0;
        }

        user.LastFailedAttemptUtc = DateTime.UtcNow;
    }
    return await base.AccessFailedAsync(userId);
}
```

Bây giờ ví dụ trên chạy đúng: ngày 01/04, khoảng cách từ lần sai trước là 90
ngày > cửa sổ → bộ đếm về 0 → lần sai này tính là lần thứ 1.

### Một chi tiết quan trọng: KHÔNG gọi `ResetAccessFailedCountAsync`

Cách viết trực giác sẽ là gọi `ResetAccessFailedCountAsync(userId)` để đặt lại
bộ đếm. **Không được làm vậy.**

Hàm đó gọi `UpdateAsync` bên trong. Mà `UpdateAsync` trong dự án này đã bị ghi
đè để chạy `gSecurity.ReWriteValue`. Gọi cả hai trong cùng một request thì
`ReWriteValue` chạy **hai lần** — chưa rõ hậu quả, và đó chính là lý do phải
tránh.

Thay vào đó, đoạn code trên **sửa thẳng thực thể EF đang được theo dõi**
(`user.AccessFailedCount = 0`). Khi `base.AccessFailedAsync` gọi `UpdateAsync`
lần duy nhất của nó, cả hai thay đổi được ghi xuống cùng lúc.

> **Nguyên tắc:** khi ghi đè một hàm khung, hãy tìm hiểu hàm gốc gọi những gì
> **trước khi** thêm lời gọi của mình. Hai lời gọi lồng nhau tới cùng một lớp
> lưu trữ là nguồn của những lỗi rất khó lần ra.

---

## 3. Một lỗi có thật đã được vá: đếm hai lần

Trong `AccountController.MobileLogin`:

```csharp
// BẢN CŨ
else if (await UserManager.GetLockoutEnabledAsync(user.Id) && validCredentials == null)
{
    await UserManager.AccessFailedAsync(user.Id);   // ← đếm lần 1
    // ...ghi thông báo lỗi...
}
// ...KHÔNG có return, chảy tiếp xuống...
var result = await SignInManager.PasswordSignInAsync(
                 user.UserName, password, true, shouldLockout: true);  // ← đếm lần 2
```

Nhánh `else if` xử lý xong nhưng **không `return`**, nên luồng chảy tiếp xuống
`PasswordSignInAsync` với `shouldLockout: true` — và hàm đó tự gọi
`AccessFailedAsync` một lần nữa.

**Hậu quả:** mỗi lần nhập sai được đếm thành 2. Cấu hình "khóa sau 5 lần sai"
trên thực tế khóa sau **3** lần (lần thứ 3 làm bộ đếm nhảy lên 6).

Cách sửa: `return null;` ngay trong nhánh đó.

### Tương tác giữa lỗi này và cửa sổ đếm

Khi cửa sổ đếm được thêm vào, thiệt hại từ lỗi trên **giảm đi** chứ không tăng:
bộ đếm bị thổi phồng chỉ tồn tại trong một cửa sổ rồi về 0, thay vì cộng dồn
mãi mãi.

Nhưng thứ **xấu đi** là khoảng cách giữa chính sách được công bố và thực tế:
trước đây quản trị viên không biết chính xác ngưỡng thật là bao nhiêu; giờ họ
nhìn màn hình thấy "5 lần trong 10 phút" và tin vào con số đó — trong khi hệ
thống khóa ở lần thứ 3.

> Một tính năng làm cho chính sách **rõ ràng hơn** cũng làm cho sai lệch trong
> chính sách đó **đáng kể hơn**.

---

## 4. Ba tham số rời nhau

Đọc liền nhau thành một câu:

> "Sai quá **`MaxInvalidPasswordAttempts`** lần trong vòng
> **`FailedAttemptWindowMinutes`** phút thì khóa **`LockoutMinutes`** phút."

Khác với cặp tham số mật khẩu ở [01](01-mat-khau-dinh-ky.md), ba tham số này
**không** bị ràng buộc "cùng có hoặc cùng trống", vì chúng độc lập về nghiệp vụ:

- Đặt ngưỡng mà không đặt thời gian khóa → hợp lệ (Identity dùng mặc định).
- Đặt cửa sổ mà không đặt ngưỡng → không gây hại.

`AdminController.KiemTraCauHinhKhoaTaiKhoan` chỉ chặn các giá trị **vô nghĩa**:

```csharp
if (st.MaxInvalidPasswordAttempts.HasValue && st.MaxInvalidPasswordAttempts.Value <= 0)
    return "Số lần nhập sai mật khẩu tối đa phải lớn hơn 0. Để trống nếu không muốn giới hạn.";
```

### Vì sao phải chặn cho bằng được giá trị 0

`MaxFailedAccessAttemptsBeforeLockout = 0` nghĩa là **khóa ngay ở lần nhập sai
đầu tiên**.

Mà ở tầng giao diện, một ô số bị **xóa trắng** rất dễ gửi về `0` thay vì `null`
— đó chính là lý do `mPortalSetting.js` phải có `allowNull: true`. Đây không
phải tình huống giả định; nó là hành vi mặc định của ExtJS.

Hai lớp bảo vệ cho cùng một lỗi:

```javascript
// mPortalSetting.js — ngăn gửi 0 đi
{ name: 'MaxInvalidPasswordAttempts', type: 'int', allowNull: true },
```
```csharp
// AdminController.cs — từ chối 0 nếu nó vẫn tới được
if (st.MaxInvalidPasswordAttempts.HasValue && st.MaxInvalidPasswordAttempts.Value <= 0) ...
```

---

## 5. Vì sao không backfill `LastFailedAttemptUtc`

`MIGRATION 004` thêm cột này **không kèm giá trị mặc định** — mọi dòng cũ đều
`NULL`.

Khác với `MIGRATION 003` (ở đó phải backfill `PasswordChangedUtc` để bảo vệ dữ
liệu cũ). Ở đây `NULL` nghĩa là "không biết lần sai trước xảy ra khi nào", và
đoạn code xử lý đúng tình huống đó: điều kiện `user.LastFailedAttemptUtc.HasValue`
không thỏa → không đặt lại bộ đếm → giữ nguyên hành vi cũ của Identity cho tới
lần sai kế tiếp, rồi từ đó trở đi cửa sổ đếm hoạt động bình thường.

Backfill bằng `GETUTCDATE()` sẽ **sai**: nó tuyên bố rằng mọi tài khoản trong
hệ thống vừa mới nhập sai mật khẩu tại thời điểm chạy migration.

---

## 6. Nơi cần biết

| Việc | Tệp |
|---|---|
| Ghi đè bộ đếm | `gPortal.Framework/Identity/IdentityConfig.cs` |
| Cấu hình Identity | `IdentityConfig.cs`, dòng ~112–130 |
| Luồng đăng nhập | `AccountController.Login`, `AccountController.MobileLogin` |
| Kiểm tra cấu hình | `AdminController.KiemTraCauHinhKhoaTaiKhoan` |
| Giao diện | `PortalWebsite.js` — tab **"Đăng nhập"** |
| Migration | `Database/DbUpdate.sql` — 004 |
