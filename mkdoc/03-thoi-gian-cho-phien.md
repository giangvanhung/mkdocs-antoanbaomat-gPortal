# 03 — Thời gian chờ của phiên làm việc

## Yêu cầu nghiệp vụ

> (a) Có chức năng cho phép thiết lập **giới hạn thời gian chờ (timeout)** để
> đóng phiên kết nối khi Phần mềm **không nhận được yêu cầu từ người dùng**.
>
> (b) **Hiển thị thông báo**, đóng phiên kết nối đã hết hạn và **yêu cầu đăng
> nhập lại**.

---

## 1. Vì sao không dùng sẵn `ExpireTimeSpan` + `SlidingExpiration`

OWIN có sẵn cơ chế này. Nhưng `SlidingExpiration` **chỉ gia hạn cookie khi đã
quá NỬA vòng đời**.

Nghĩa là với `ExpireTimeSpan = 15 phút`:

``` title="Ví dụ minh họa SlidingExpiration — không phải mã nguồn"
Người dùng hoạt động ở phút thứ 7   →  chưa quá nửa  →  KHÔNG gia hạn
                                        →  cookie vẫn hết hạn ở phút 15
                                        →  thời gian không hoạt động thực tế: 8 phút

Người dùng hoạt động ở phút thứ 8   →  đã quá nửa    →  gia hạn
                                        →  thời gian không hoạt động thực tế: 15 phút
```

Cấu hình "15 phút" mà hệ thống đóng phiên **ở đâu đó trong khoảng 7–15 phút**
thì không thể gọi là "thiết lập giới hạn thời gian chờ". Con số quản trị viên
nhập vào không tương ứng với hành vi thật.

### Cách làm ở đây

Tự ghi mốc hoạt động, nên sai số bị chặn ở **độ hạt** cố định thay vì phụ thuộc
vào tổng thời gian chờ:

```csharp title="gPortal_Portal/gPortal/Authorization/SessionTimeoutGuard.cs:202-206"
private static double DoHatGiay(int phut)
{
    double motPhanTu = phut * 60.0 / 4.0;
    return Math.Min(60.0, motPhanTu);
}
```

Bình thường sai số tối đa 60 giây. Với thời gian chờ rất ngắn thì lấy một phần
tư — để cấu hình 1 phút không biến thành "đóng phiên đâu đó trong khoảng 1 đến
2 phút".

---

## 2. Mốc hoạt động cất ở đâu — và vì sao

| Nơi cất | Vì sao KHÔNG chọn |
|---|---|
| Bảng CSDL | Mốc này cập nhật cực kỳ thường xuyên mà **chỉ mỗi phiên đó cần biết**. Ghi CSDL mỗi request là cái giá quá đắt. |
| `Session` của ASP.NET | **`Session` chưa sẵn sàng** ở giai đoạn `PostAuthenticateRequest` — mà đó lại đúng là nơi chốt chặn phải nằm để phủ được cả trang `.aspx`. |
| ✅ **Vé xác thực OWIN** | Đi theo request, không tốn CSDL, và **đã được OWIN ký + mã hóa** nên người dùng không sửa được giá trị bên trong. |

```csharp title="gPortal_Portal/gPortal/Authorization/SessionTimeoutGuard.cs:43 và :248-249"
private const string KhoaMocHoatDong = "gp:lastActivityUtc";
context.Properties.Dictionary[KhoaMocHoatDong] = moc.Ticks.ToString(...);
```

---

## 3. Hai nửa, hai chỗ

``` title="Sơ đồ hai nửa — không phải mã nguồn"
┌───────────────────────────────────────────────────────────────┐
│  Startup.Auth.cs  →  OnValidateIdentity                       │
│  ─────────────────────────────────────────────                │
│  SessionTimeoutGuard.HetHanVaDongPhien(context)               │
│    • Quyết định phiên còn sống hay không                      │
│    • Đẩy mốc hoạt động                                        │
│    • KHÔNG đụng vào Response                                  │
└───────────────────────────────────────────────────────────────┘
                              ↓ đặt cờ "gp:sessionExpired"
┌───────────────────────────────────────────────────────────────┐
│  Global.asax  →  Application_PostAuthenticateRequest          │
│  ─────────────────────────────────────────────                │
│  SessionTimeoutGuard.Handle(context)                          │
│    • CHỈ lo hình dạng phản hồi                                │
│    • AJAX → 401 JSON;  thường → chuyển hướng                  │
└───────────────────────────────────────────────────────────────┘
```

Phải tách vì: tầng OWIN **không biết gì** về quy ước AJAX của portal, còn tầng
`Global.asax` thì **không xen được** vào lúc vé xác thực đang được kiểm.

### Khi hết giờ thì dọn những gì

```csharp title="gPortal_Portal/gPortal/Authorization/SessionTimeoutGuard.cs:89-97"
context.RejectIdentity();                                      // 1. bỏ danh tính OWIN
context.OwinContext.Authentication.SignOut(...);               // 2. gỡ cookie OWIN
DonCookiePhuTro();                                             // 3. gỡ vé FormsAuthentication
```

Bước 3 là chỗ đã phát hiện một **lỗ rò có sẵn** trong dự án:
`CookieHelper.AddCookieAuth` phát thêm một vé `FormsAuthentication` mang theo
tên đăng nhập và vai trò, nhưng `CookieHelper.ClearCookieAuthen` **không dọn vé
đó**. "Đóng phiên" khi ấy mới chỉ đúng một nửa — dấu vết xác thực vẫn nằm trong
trình duyệt thêm một ngày nữa.

Đã bổ sung:

```csharp title="gPortal_Portal/gPortal.Framework/CookieHelper.cs:191-192"
System.Web.Security.FormsAuthentication.SignOut();
ClearCookie(System.Web.Security.FormsAuthentication.FormsCookieName);
```

Vì `ClearCookieAuthen` được dùng chung bởi cả đường "hết giờ" lẫn đường "tự
đăng xuất", sửa một chỗ là cả hai đường cùng đúng — hai bản dọn viết tay sẽ có
ngày lệch.

---

## 4. Đồng hồ thứ hai: phía client

### Vì sao cần thêm đồng hồ nữa

Đồng hồ máy chủ đếm theo **request**. Nhưng các ứng dụng ExtJS trong portal
**gọi AJAX liên tục ở nền**. Kết quả:

> Máy chủ gần như **không bao giờ** thấy phiên "không hoạt động", dù người dùng
> đã rời khỏi máy từ lâu.

Máy chủ không phân biệt được "đang làm việc" với "tab bị bỏ quên đang tự poll".
Chuột và bàn phím thì phân biệt được.

### Nguyên tắc ghép hai đồng hồ

> **Cả hai chỉ được phép KẾT THÚC phiên. Không đồng hồ nào được phép KÉO DÀI.
> Cái nào hết trước thì thắng.**

``` title="Công thức hạn hết giờ — không phải mã nguồn"
   hạn hết giờ = min(mocRequest, mocThaoTac) + thoiGianCho
```

Điều này khiến đồng hồ client chỉ làm luật **chặt hơn**, không bao giờ lỏng
hơn. Và nó tôn trọng đúng chữ nghĩa của yêu cầu:

| Hành động | Có phải "yêu cầu từ người dùng"? |
|---|---|
| Di chuột, gõ phím, cuộn trang | ❌ Không tạo ra request nào |
| **Bấm nút "Tiếp tục làm việc"** | ✅ Có — một cú bấm nút là một yêu cầu thật |

Vì vậy `gp-session-timeout.js` **không bao giờ tự gọi về máy chủ để gia hạn**.
Muốn gia hạn thì phải bấm nút.

### Vì sao chỉ theo dõi thao tác thì sai

Người dùng di chuột 30 phút mà không gửi request nào → máy chủ đã đóng phiên từ
lâu, client vẫn tưởng còn sống → họ bấm một cái và bị đá ra **không có lời cảnh
báo nào**. Lấy mốc sớm hơn thì hai đồng hồ luôn khớp nhau.

### Hai chi tiết kỹ thuật đáng nhớ

**a. Phải nghe ở pha CAPTURE, không phải pha bubble:**

```javascript title="gPortal_Portal/gPortal/Scripts/gp-session-timeout.js:85"
document.addEventListener(loai[i], ghiNhanThaoTac, { capture: true, passive: true });
```

Hai lý do, cả hai đều bắt buộc:
1. ExtJS gọi `stopPropagation()` ở khá nhiều chỗ (kéo thả, chọn ô trong grid,
   menu ngữ cảnh). Nghe ở pha bubble thì đúng những thao tác **tích cực nhất**
   lại không tới được `document`.
2. Sự kiện `scroll` **không bubble**. Ở pha bubble thì cuộn trang không bao giờ
   được ghi nhận.

**b. Bỏ qua thao tác thụ động khi hộp cảnh báo đang hiện:**

```javascript title="gPortal_Portal/gPortal/Scripts/gp-session-timeout.js:60-63"
function ghiNhanThaoTac() {
    if (dangHienHop || daHetGio) return;
    mocThaoTac = Date.now();
}
```

Nếu không, chỉ cần rê chuột là hộp tự biến mất **mà không có request nào được
gửi** — đồng hồ client lùi lại trong khi đồng hồ máy chủ vẫn chạy tới, và người
dùng bị đá ra ngay sau đó dù màn hình vừa báo là ổn.

---

## 5. Nhúng script vào MỌI trang — hai đường

Portal có **24 trang `.aspx`** dùng **4 master page** khác nhau, và **10 trang
không có master nào**. Thêm thẻ `<script>` bằng tay là 14 chỗ, và trang mới
thêm sau này chắc chắn sẽ bị quên.

| Loại trang | Cách nhúng |
|---|---|
| MVC | `Views/Shared/_SessionTimeout.cshtml`, chèn vào cả 7 `_Layout*.cshtml` |
| WebForms | `Global.asax` → `Application_PreRequestHandlerExecute`, móc `Page.PreRenderComplete` |

Đường thứ hai phủ **mọi** handler là `Page`, không cần biết nó dùng master gì:

```csharp title="gPortal_Portal/gPortal/Global.asax.cs:86-116 (rút gọn)"
protected void Application_PreRequestHandlerExecute(object sender, EventArgs e)
{
    var page = HttpContext.Current.Handler as System.Web.UI.Page;
    if (page == null) return;
    page.PreRenderComplete += (s, ev) => { /* đăng ký script */ };
}
```

Phải là `PreRequestHandlerExecute` vì tới đây `HttpContext.Handler` đã được
dựng (nên biết request này có phải một trang hay không), mà vẫn còn sớm hơn
vòng đời của `Page` nên kịp đăng ký script.

### Không có iframe — và điều đó rất quan trọng

Đã kiểm tra: **cả 24 trang `.aspx` và mọi `View.ascx` đều không dùng iframe.**
Các ứng dụng ExtJS được nhúng thẳng vào chính những trang đó.

Nghĩa là chúng **dùng chung một `document`** với trang chủ. Một listener duy
nhất ở pha capture trên `document` nghe được cả thao tác bên trong ứng dụng
ExtJS — **không phải sửa gì trong `gPortal_gClient`.**

Nếu có iframe thì mọi thứ sẽ khác hẳn: mỗi iframe là một `document` riêng, một
`window` riêng, và script ở trang cha không nghe được sự kiện bên trong nó.

---

## 6. Nơi cần biết

| Việc | Tệp |
|---|---|
| Đồng hồ máy chủ | `gPortal/Authorization/SessionTimeoutGuard.cs` |
| Móc vào OWIN | `gPortal/App_Start/Startup.Auth.cs` |
| Đồng hồ client | `gPortal/Scripts/gp-session-timeout.js` |
| Nhúng cho MVC | `Views/Shared/_SessionTimeout.cshtml` |
| Nhúng cho WebForms | `Global.asax.cs` → `Application_PreRequestHandlerExecute` |
| API chính sách | `AccountController.SessionPolicy` |
| Thông báo | `Views/Account/Login.cshtml` (`?expired=1`) |
| Migration | `Database/DbUpdate.sql` — 005, 006 |

---

## 7. Chẩn đoán khi "không thấy nó tự đăng xuất"

Đây là câu hỏi hay gặp nhất, và nó thường không phải lỗi.

### Trước hết: máy chủ KHÔNG có job nền

Máy chủ không "đăng xuất" ai cả trong lúc bạn đang ngồi chờ. Nó **quyết định
tại thời điểm request kế tiếp**. Không có request thì không có gì xảy ra — và
cũng không cần, vì không ai đang dùng hệ thống.

Hệ quả:

| Cách thử | Đang thử cái gì |
|---|---|
| Đóng hết trình duyệt, chờ 3 phút, mở lại một trang | **Đồng hồ máy chủ** |
| Ngồi mở trang, không đụng vào, chờ | **Đồng hồ client** |

### Nếu ngồi mở trang admin mà không thấy gì

Đó chính là tình huống mà đồng hồ client sinh ra để xử lý — vì ExtJS đang gọi
AJAX ở nền nên **đồng hồ máy chủ sẽ không bao giờ hết giờ**. Nếu cũng không
thấy hộp cảnh báo, kiểm tra theo thứ tự:

```javascript title="Gõ trong DevTools Console — biến đặt ở gp-session-timeout.js:40-41"
// Mở DevTools → Console, gõ:
gpSessionTimeoutDaChay
```

| Kết quả | Nghĩa là | Xem tiếp |
|---|---|---|
| `undefined` | Script **chưa được nạp** vào trang này | Kiểm tra thẻ `<script>` trong View Source |
| `true` | Script đã chạy | Sang bước dưới |

Nếu script đã chạy, mở tab **Network**, tìm lời gọi `Account/SessionPolicy` và
xem phần trả về:

```json title="Phản hồi của GET /Account/SessionPolicy"
{ "soPhut": 2, "soPhutCanhBao": 1, "thongBao": "...", "urlHetGio": "..." }
```

| Trả về | Nghĩa là |
|---|---|
| `soPhut: 0` | **Cấu hình chưa được lưu xuống CSDL** — kiểm tra bằng SQL bên dưới |
| Không có lời gọi nào | Script không chạy được — xem Console có lỗi JS không |
| `soPhut: 2` mà vẫn không có gì | Đây mới thật sự là lỗi cần đào tiếp |

```sql title="Chạy trong SQL Server Management Studio"
SELECT SessionTimeoutMinutes, SessionWarnBeforeMinutes
FROM dbo.gp_PortalSettings;
```

**Lưu ý:** `SessionWarnBeforeMinutes` phải **nhỏ hơn hẳn**
`SessionTimeoutMinutes`. Với timeout = 2 phút thì cảnh báo phải là 1. Nếu để
trống thì không có hộp cảnh báo, nhưng **vẫn phải tự đăng xuất** khi hết 2 phút.
