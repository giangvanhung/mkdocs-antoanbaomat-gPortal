# 04 — Giới hạn địa chỉ mạng quản trị

## Yêu cầu nghiệp vụ

> (a) Có **giao diện** cho phép quản trị viên quản lý chính sách về giới hạn
> địa chỉ mạng quản trị được phép truy cập, quản trị Phần mềm từ xa.
>
> (b) Có chức năng **thực thi** chính sách đó.

---

## 1. Ý tưởng một câu

> Mật khẩu trả lời câu hỏi **"anh là ai"**. Tính năng này trả lời câu hỏi
> **"anh đang ngồi ở đâu"**.

Kẻ trộm được mật khẩu quản trị vẫn không vào được, nếu hắn không ngồi trong
mạng được phép. Đây là lớp phòng thủ độc lập với mọi lớp còn lại — nó không
quan tâm mật khẩu mạnh hay yếu.

---

## 2. Mô hình dữ liệu

Bảng mới `gp_AdminIpRules`:

| Cột | Ý nghĩa |
|---|---|
| `Id` | Khóa chính |
| `IpValue` | Nội dung quy tắc (xem 3 dạng bên dưới) |
| `Description` | Ghi chú: dải này của ai, mở ra vì việc gì |
| `Enabled` | Tắt tạm một dòng mà không mất ghi chú |
| `CreatedUtc`, `CreatedBy` | Ai thêm, khi nào |

### Vì sao là bảng riêng chứ không phải một cột trong `gp_PortalSettings`

Chính sách này là một **danh sách** dài không biết trước. Nhét vào một cột văn
bản ngăn cách bằng dấu phẩy thì:

- Mất chỗ ghi chú → sáu tháng sau không ai biết dòng nào còn cần thiết.
- Mất khả năng tắt tạm một mục.
- Mỗi lần sửa là **ghi đè cả chuỗi** → hai quản trị viên sửa cùng lúc sẽ xóa
  mất phần của nhau.

### Ba dạng quy tắc được chấp nhận

| Dạng | Ví dụ IPv4 | Ví dụ IPv6 |
|---|---|---|
| Một địa chỉ | `192.168.1.10` | `::1` |
| Dải CIDR | `192.168.1.0/24` | `2001:db8::/32` |
| Khoảng | `192.168.1.1-192.168.1.50` | — |

**Không hỗ trợ tên miền.** Tên miền phải tra DNS mới ra địa chỉ, mà DNS thì kẻ
tấn công có thể tác động được và kết quả đổi theo thời gian — một chốt chặn phụ
thuộc DNS là chốt chặn mà **người khác cầm chìa khóa**.

---

## 3. Quy ước quan trọng nhất: BẢNG RỖNG = TẮT TÍNH NĂNG

Không có dòng nào đang bật → **cho phép mọi địa chỉ**.

Đây là lựa chọn có cân nhắc, không phải sơ suất. Nếu quy ước ngược lại — "rỗng
thì cấm tất" — thì **đúng khoảnh khắc script migration chạy xong, toàn bộ quản
trị viên bị khóa khỏi hệ thống**, và không còn đường nào vào để thêm dòng đầu
tiên. Cách sửa duy nhất là vào tận máy chủ chạy SQL bằng tay.

Vì cùng lý do đó, `MIGRATION 007` **không chèn dữ liệu mẫu nào cả**.

Cũng vì vậy không cần một "công tắc bật/tắt" riêng: danh sách rỗng **chính là**
công tắc tắt. Thêm một công tắc nữa sẽ tạo ra trạng thái nguy hiểm "công tắc
bật nhưng danh sách rỗng" = khóa toàn bộ.

---

## 4. Phạm vi áp dụng: hai vế bù cho nhau

Một request bị soi nếu **thỏa một trong hai**:

``` title="Hai vế phạm vi áp dụng — không phải mã nguồn"
(1) Đường dẫn nằm trong bề mặt quản trị:  /admin , /gclient/gadmin
    → kể cả khi CHƯA đăng nhập

(2) Người đang đăng nhập là quản trị viên
    → dù họ đang mở trang nào
```

| Vế | Bắt được cái mà vế kia bỏ sót |
|---|---|
| (1) | **Khách chưa đăng nhập** — vế (2) không thấy họ |
| (2) | **Đường dẫn mới** — danh sách ở vế (1) chắc chắn sẽ thiếu khi có người thêm chức năng |

Vế (1) chặn **trước cả khi xác thực**. Đó là mức mạnh hơn: kẻ ở mạng ngoài
không chạm được tới form đăng nhập quản trị, nên không có gì để dò mật khẩu.

Vế (2) là bảo hiểm cho sự lười biếng trong tương lai. Không có nó thì chỉ cần
một endpoint quản trị nằm ngoài danh sách đường dẫn là cả chính sách vô hiệu.

### Ngoại lệ luôn cho qua

```csharp title="gPortal_Portal/gPortal/Authorization/AdminIpGuard.cs:87-92"
private static readonly string[] DuongDanChoQua =
{
    "/account/login", "/account/logoff", "/manage/logoff"
};
```

Bắt buộc phải có đường đăng xuất. Bị chặn ở **mọi** request nghĩa là bị chặn cả
ở nút Đăng xuất, và lối thoát duy nhất còn lại là tự xóa cookie trong trình
duyệt. Không ai nên phải làm việc đó.

---

## 5. Chặn ở HAI nơi, vì hai lý do khác nhau

### a. Chốt từng request — `AdminIpGuard.Handle` trong `Global.asax`

Đây là chốt thật, phủ mọi đường: MVC, WebForms, API, cả `MobileLogin`.

### b. Chốt lúc đăng nhập — `AccountController.RedirectAfterSignIn`

Vì sao vẫn cần, khi đã có (a)?

Tới thời điểm `RedirectAfterSignIn` chạy, `PasswordSignInAsync` **đã phát ra
cookie xác thực** và nó đang nằm trong trình duyệt. Chốt (a) sẽ chặn mọi lần
dùng nó **trên máy này**, nhưng bản thân cái cookie vẫn là một **vé hợp lệ** —
mang sang một mạng khác là dùng được.

Thu hồi ngay tại đây thì kẻ có mật khẩu đúng nhưng ngồi sai chỗ **không nhận
được gì mang đi cả**:

```csharp title="gPortal_Portal/gPortal/Controllers/AccountController.cs:1069-1072"
AuthenticationManager.SignOut();
CookieHelper.ClearCookieAuthen();
return RedirectToAction("Login", "Account", new { ipdenied = 1 });
```

### Vì sao đặt ở `RedirectAfterSignIn` chứ không ở từng action đăng nhập

Đó là chỗ **duy nhất** mà cả bốn đường đăng nhập web đều đi qua (đăng nhập
thường ×2 nhánh, SSO qua `LoginPage`, xác thực hai lớp qua `VerifyCode`). Vá
từng đường một là cách chắc chắn để bỏ sót đúng một đường.

### Một cái bẫy trong đó

```csharp title="gPortal_Portal/gPortal/Controllers/AccountController.cs:1034-1040"
// KHÔNG dùng gPortalUser.IsAdmin ở đây!
laQuanTri = UserManager.IsInRole(user.Id, Role.Administrator) || ...
```

`gPortalUser.IsAdmin` đọc `HttpContext.Current.User`. Nhưng tại thời điểm này,
`User` của request hiện tại **vẫn là người cũ (chưa đăng nhập)** — vé mới chỉ
có hiệu lực từ request sau. Dùng nó ở đây thì mọi quản trị viên đều "không phải
quản trị" và chốt không bao giờ kích hoạt.

---

## 6. Lấy địa chỉ client — chỗ dễ sai nhất

### Mặc định: CHỈ tin địa chỉ TCP

```csharp title="gPortal_Portal/gPortal.Framework/Security/DiaChiClient.cs:40"
IPAddress diaChiTcp = KhopDiaChiMang.Doc(request.UserHostAddress);
```

**Không** tin `X-Forwarded-For`. Tiêu đề đó do client tự đặt: tin nó vô điều
kiện thì chỉ cần gửi kèm `X-Forwarded-For: 192.168.1.10` là qua chốt, và cả
tính năng chỉ còn là đồ trang trí.

### Nhưng nếu portal đứng sau proxy?

Địa chỉ TCP khi ấy luôn là của proxy → **mọi người dùng trông giống hệt nhau**
→ chính sách thành "cho tất cả" hoặc "cấm tất cả".

Lời giải: khai báo proxy tin cậy trong `Web.config`. **Chỉ khi** địa chỉ TCP
đúng là một proxy đã khai báo, ta mới đọc `X-Forwarded-For`:

```xml title="gPortal_Portal/gPortal/Web.config:63 (đang bị chú thích)"
<add key="gp:AdminIpTrustedProxies" value="10.0.0.5, 10.0.0.6" />
```

Và đọc mục **phải cùng bên phải** của chuỗi, vì proxy nối thêm vào cuối — các
mục bên trái do client gửi lên nên bịa được.

### Vì sao KHÔNG dùng `WebSecurity.getIpAdress()` có sẵn

Hàm đó trong `Global.asax.cs`:

```csharp title="gPortal_Portal/gPortal/Global.asax.cs:192-199"
public static string getIpAdress()
{
    string ipclient = HttpContext.Current.Request.QueryString.Get("ip");
    if (!string.IsNullOrEmpty(ipclient)) return ipclient;      // ← đọc ?ip= TRƯỚC
    return HttpContext.Current.Request.UserHostAddress;
}
```

Nó đọc `?ip=` từ query string **trước** khi đọc địa chỉ thật. Với việc hiển thị
thông báo thì vô hại. Dùng cho chốt chặn thì **bất kỳ ai cũng vượt qua** bằng
cách thêm `?ip=192.168.1.10` vào URL.

Hàm đó vẫn giữ nguyên (đang được dùng cho thông báo chặn SQL injection), chỉ là
không được dùng lại cho mục đích này.

---

## 7. Ba lớp chống tự khóa

Đây là tính năng **duy nhất** trong hệ thống có thể khóa chính người vừa cấu
hình nó. Nên phải có đường lùi.

### Lớp 1 — Khi LƯU: từ chối nếu bạn sẽ tự khóa mình

```csharp title="gPortal_Portal/gPortal/Controllers/AdminController.cs:1054-1077 (rút gọn)"
private static string KiemTraKhongTuKhoa(hienCo, cu, moi)
{
    // Dựng ra bộ quy tắc SẼ có sau thao tác này
    var sauKhiLuu = hienCo.Where(x => cu == null || x.Id != cu.Id)
                          .Where(x => x.Enabled)
                          .Select(x => x.IpValue).ToList();
    if (moi != null && moi.Enabled) sauKhiLuu.Add(moi.IpValue);

    // Rồi hỏi ĐÚNG câu mà chốt chặn sẽ hỏi ở request tiếp theo
    if (AdminIpGuard.DuocPhep(diaChi, sauKhiLuu)) return null;

    return "Không lưu được: sau thay đổi này, chính địa chỉ bạn đang dùng (...) "
         + "sẽ không còn được phép truy cập...";
}
```

Điểm mấu chốt: **không dự đoán, không suy luận song song**. Nó gọi lại chính
hàm `AdminIpGuard.DuocPhep` mà chốt chặn dùng, nên câu trả lời ở đây **không
thể lệch** với hành vi thật.

Áp dụng cho **cả xóa lẫn sửa**: xóa đúng dòng đang bảo vệ mình trong khi vẫn
còn dòng khác cũng là tự đuổi mình ra ngoài.

Và vì bộ quy tắc rỗng khiến `DuocPhep` trả về `true`, **xóa dòng cuối cùng luôn
được phép** — đó chính là cách tắt tính năng.

### Lớp 2 — Khi BỊ CHẶN: nói rõ địa chỉ đang dùng

Trang 403 hiện đúng con số. Đây không phải rò rỉ thông tin — bất kỳ trang "what
is my IP" nào cũng nói được điều đó. Đổi lại, nó là khác biệt giữa một tính
năng dùng được và một tính năng mà mỗi lần trục trặc là một cuộc gọi hỗ trợ mò
mẫm.

Giao diện cũng hiện sẵn địa chỉ của bạn ngay trên thanh công cụ của lưới
(`/Admin/MyIp`), để không phải đoán.

### Lớp 3 — Khi ĐÃ LỠ KHÓA: công tắc trong `Web.config`

```xml title="gPortal_Portal/gPortal/Web.config:62 (đang bị chú thích)"
<add key="gp:AdminIpRestrictionDisabled" value="true" />
```

Sửa được file này nghĩa là đã kiểm soát máy chủ, nên đây **không phải lỗ hổng
mới** — nó chỉ là đường lùi bắt buộc phải có. Dùng khi địa chỉ IP của nhà mạng
thay đổi ngoài dự kiến.

---

## 8. Hướng hỏng: khác với hai chốt kia

Hai chốt còn lại **cho qua** khi CSDL lỗi. Chốt này thì không, và đó là chủ ý:

| Chốt | "Cho qua" nghĩa là gì |
|---|---|
| Mật khẩu, phiên làm việc | Người **đã đăng nhập hợp lệ** được làm việc tiếp |
| **Địa chỉ mạng** | **Cấp thêm quyền** cho một địa chỉ lẽ ra bị cấm |

Vế thứ hai nặng hơn hẳn. Cách xử lý:

```csharp title="gPortal_Portal/gPortal/Authorization/AdminIpGuard.cs:138-144"
catch (Exception ex)
{
    gPortalLogger._log.Error(...);
    // Giữ nguyên bộ quy tắc CŨ nếu đã từng đọc được
    _hetHanUtc = DateTime.UtcNow.AddSeconds(10);
}
```

- CSDL trục trặc thoáng qua → **dùng lại bộ quy tắc cũ trong bộ nhớ** → chính
  sách vẫn có hiệu lực.
- Chưa bao giờ đọc được lần nào (khởi động mà CSDL đã hỏng) → cho qua, và ghi
  log `Error`. Lúc này portal cũng đang không phục vụ được gì; chặn thêm chỉ
  khóa nốt người có thể vào sửa.

Bộ quy tắc được nhớ **60 giây**. Trên chính máy chủ vừa sửa thì không có độ trễ
nào, vì `AdminController` gọi `AdminIpGuard.XoaBoNho()` ngay sau khi lưu. 60
giây chỉ là độ trễ tối đa khi chạy nhiều máy chủ.

---

## 9. Một cái bẫy trong việc đọc địa chỉ IP

`IPAddress.TryParse` của .NET **chấp nhận dạng thiếu đoạn và tự suy diễn**:

``` title="Ví dụ minh họa IPAddress.TryParse — không phải mã nguồn"
"192.168.1"  →  192.168.0.1     (!)
"10"         →  0.0.0.10        (!)
```

Nghĩa là một quy tắc gõ thiếu sẽ được **nhận**, nhưng nó bảo vệ một địa chỉ
**khác** với địa chỉ người nhập nghĩ — kiểu sai tệ nhất, vì màn hình vẫn báo
lưu thành công.

`KhopDiaChiMang.Doc` siết thêm: dạng IPv4 phải có **đúng 4 đoạn**, mỗi đoạn là
số nguyên 0–255.

(Không áp dụng cho IPv6, vì IPv6 có cú pháp rút gọn hợp lệ như `::1`.)

### Và một bẫy nữa: IPv4 đi qua tầng IPv6

Trên IIS/Windows rất hay gặp: địa chỉ `192.168.1.5` tới nơi dưới dạng
`::ffff:192.168.1.5`. So bằng byte thì **khác hoàn toàn**.

Không quy chuẩn thì quản trị viên nhập đúng địa chỉ của mình mà vẫn bị chặn, và
sẽ không hiểu vì sao:

```csharp title="gPortal_Portal/gPortal.Framework/Security/KhopDiaChiMang.cs:37-46"
public static IPAddress ChuanHoa(IPAddress diaChi)
{
    if (diaChi.AddressFamily == AddressFamily.InterNetworkV6 && diaChi.IsIPv4MappedToIPv6)
        return diaChi.MapToIPv4();
    return diaChi;
}
```

> **Lưu ý khi thử trên localhost:** trình duyệt thường tới qua `::1` (IPv6
> loopback), **không** phải `127.0.0.1`. Hai địa chỉ này là khác nhau và không
> tự quy đổi cho nhau. Cứ nhìn con số mà giao diện hiện ra rồi thêm đúng con số
> đó.

---

## 10. Nơi cần biết

| Việc | Tệp |
|---|---|
| Chốt chặn | `gPortal/Authorization/AdminIpGuard.cs` |
| Luật so khớp IP | `gPortal.Framework/Security/KhopDiaChiMang.cs` |
| Lấy địa chỉ client (`X-Forwarded-For`) | `gPortal.Framework/Security/DiaChiClient.cs` |
| Chặn lúc đăng nhập | `AccountController.ChanQuanTriSaiDiaChi` — `AccountController.cs:1053-1074` |
| API quản lý | `AdminController` — vùng `#region "AdminIpRules"` |
| Chống tự khóa | `AdminController.KiemTraKhongTuKhoa` — `AdminController.cs:1054-1077` |
| Giao diện | `PortalWebsite.js` — tab **"Địa chỉ quản trị"** |
| Cấu hình hạ tầng | `gPortal/Web.config` — `gp:AdminIpRestrictionDisabled`, `gp:AdminIpTrustedProxies` |
| Migration | `Database/DbUpdate.sql` — 007 |
