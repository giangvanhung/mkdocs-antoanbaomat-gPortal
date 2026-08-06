# 09 — SMTP: vì sao không gửi được và đã sửa gì

Nền cho việc bật **xác thực hai lớp qua email (2FA)**: không gửi được thư thì
không có 2FA.

---

## 1. Tóm tắt: bốn lỗi độc lập, chồng lên nhau

Không phải một nguyên nhân duy nhất. Có **bốn** lỗi riêng biệt, mỗi cái đủ để
chặn việc gửi thư, và chúng che lấp lẫn nhau:

| # | Lỗi | Hậu quả |
|---|---|---|
| 1 | Câu truy vấn lưu cấu hình **không dịch được sang SQL** | Cấu hình SMTP **chưa bao giờ được lưu** |
| 2 | Khóa chưa tồn tại thì **bỏ qua trong im lặng** | Trên CSDL mới, lưu xong vẫn trống |
| 3 | Giao diện gọi `ajaxPUT` **không có callback** | Lỗi ở (1) và (2) **bị nuốt sạch** |
| 4 | **TLS 1.2 không được bật** | Kể cả khi cấu hình đúng, kết nối vẫn hỏng |

Điểm khiến việc này khó lần ra: lỗi (3) làm cho lỗi (1) và (2) **không để lại
dấu hiệu nào**. Người dùng bấm Cập nhật, thấy chữ "Cập nhật thành công", và
hoàn toàn hợp lý khi tin rằng cấu hình đã được lưu. Từ đó mọi suy đoán về sau
đều đi sai hướng.

---

## 2. Lỗi 1 — Câu truy vấn không dịch được sang SQL

`gp_PortalSettingsExStore.Update(List<>)`, bản cũ:

```csharp
var e = this.DbEntitySet
            .Where(x => x.KeySetting.Equals(keysettings, StringComparison.CurrentCultureIgnoreCase))
            .FirstOrDefault();
```

`DbEntitySet` là `IQueryable` — câu `Where` này phải được **dịch sang SQL**.
Entity Framework **không dịch được** `String.Equals` có tham số
`StringComparison`, vì SQL không có khái niệm tương đương.

Kết quả: `NotSupportedException` ngay khi chạm cơ sở dữ liệu.

> `LINQ to Entities does not recognize the method`
> `'Boolean Equals(System.String, System.StringComparison)'`

Đây không phải lỗi thỉnh thoảng mới xảy ra. Nó **luôn luôn** xảy ra, nghĩa là
hàm lưu cấu hình mở rộng — trong đó có toàn bộ cấu hình SMTP — **chưa bao giờ
chạy được**.

### Đã sửa

```csharp
var khoaThuong = khoa.ToLower();
var e = this.DbEntitySet
            .Where(x => x.KeySetting.ToLower() == khoaThuong)
            .FirstOrDefault();
```

`ToLower()` thì EF dịch được thành `LOWER()` trong SQL, và cách so sánh này
không phụ thuộc vào collation của cơ sở dữ liệu.

> **Bài học:** trong LINQ, `IQueryable` và `IEnumerable` trông giống hệt nhau
> khi đọc code nhưng chạy ở hai nơi khác nhau. Cái chạy trên C# thì mọi hàm
> .NET đều dùng được; cái phải dịch sang SQL thì không. Trình biên dịch **không
> bắt được** khác biệt này — nó chỉ nổ lúc chạy.

---

## 3. Lỗi 2 — Khóa chưa tồn tại thì bỏ qua trong im lặng

Bản cũ chỉ có một nhánh:

```csharp
if (e != null)
{
    e.ValueSetting = ...;
    Context.Entry(e).State = EntityState.Modified;
}
// KHÔNG có else
```

Trên một cơ sở dữ liệu chưa từng có dòng `SMTPHost` trong `gp_PortalSettingsEx`,
quản trị viên điền đầy đủ, bấm Cập nhật, màn hình báo thành công — và **không
có gì được lưu**.

Sau đó `EmailService` đọc ra `null` rồi báo *"Không có cấu hình SMTP"*, ở một
chỗ hoàn toàn khác và muộn hơn nhiều.

### Đã sửa — chuyển thành upsert

```csharp
else
{
    this.DbEntitySet.Add(new gp_PortalSettingsEx
    {
        Id = Guid.NewGuid().ToString(),
        KeySetting = khoa,
        ValueSetting = entity[i].ValueSetting,
        GroupName = entity[i].GroupName
    });
}
```

> **Bài học:** một `if` không có `else` là một quyết định. Nếu quyết định đó là
> "không làm gì", nó phải được viết ra — hoặc bằng nhánh `else` có chú thích,
> hoặc bằng cách xử lý luôn. Bỏ trống là để người đọc sau tự đoán, và họ sẽ
> đoán rằng trường hợp đó không xảy ra được.

---

## 4. Lỗi 3 — Giao diện nuốt lỗi

`PortalWebsite.js`, bản cũ:

```javascript
ajaxPUT('/Admin/PortalSettingsEx', lst)     // ← không có tham số thứ ba
```

`ajaxPUT(url, data, callback)` — thiếu `callback` thì **cả nhánh thành công lẫn
nhánh lỗi đều không làm gì**.

Trong khi đó, lời gọi **trước** nó (`ajaxPOST /Admin/PortalSettings`) thành
công và hiện toast **"Cập nhật thành công"**.

Người dùng thấy đúng một thông báo, và thông báo đó nói về một việc khác.

### Đã sửa

Thêm callback hiện hộp lỗi rõ ràng khi lưu cấu hình mở rộng thất bại. Đồng thời
`UpdatePortalSettingsEx` phía máy chủ được bọc `try/catch` để trả về `code=100`
kèm câu lỗi, thay vì để ngoại lệ thành HTTP 500.

> **Bài học:** đây là ví dụ sạch nhất trong toàn bộ dự án về việc **báo thành
> công cho một việc không xảy ra**. Nó tệ hơn báo lỗi rất nhiều: báo lỗi thì
> người ta đi sửa, báo thành công sai thì người ta đi tìm nguyên nhân ở mọi nơi
> **trừ** chỗ thật sự hỏng.

---

## 5. Lỗi 4 — TLS 1.2 không được bật

Đây là lỗi sẽ lộ ra **sau khi** ba lỗi trên đã sửa xong.

Dự án chạy **.NET Framework 4.5.2**, và `Web.config` khai
`httpRuntime targetFramework="4.5.2"`. Với mức đó, .NET đặt
`ServicePointManager.SecurityProtocol` mặc định là `Ssl3 | Tls` — tức **TLS
1.0**.

Gmail, Office 365 và hầu hết máy chủ thư hiện nay **đã tắt TLS 1.0 và 1.1**.
Bắt tay thất bại ngay ở tầng vận chuyển, trước cả khi tới bước xác thực.

### Vì sao lỗi này đặc biệt khó chẩn đoán

Thông báo nhận được nghe **không liên quan gì tới TLS**:

> *The remote certificate is invalid*
> *Unable to read data from the transport connection*
> *Kết nối bị đóng*

Đọc những câu đó, phản xạ tự nhiên là đi kiểm **tường lửa và cổng** — trong khi
nguyên nhân nằm ở **phiên bản giao thức**.

### Đã sửa

```csharp
// DichVuThu.BatGiaoThucHienDai()
ServicePointManager.SecurityProtocol |=
    SecurityProtocolType.Tls12 | SecurityProtocolType.Tls11;
```

Hai chi tiết trong hai dòng này:

1. **`|=` chứ không phải `=`.** Các thư viện khác trong cùng tiến trình có thể
   đã bật thêm giao thức mà chúng cần; gán đè là âm thầm tắt của người khác.
2. **Không có `Tls13`.** Hằng số đó chỉ tồn tại từ .NET 4.8 — viết vào là lỗi
   biên dịch trên 4.5.2.

Gọi ở `Application_Start`, **và** gọi lại ở mỗi lần gửi thư: IIS có thể tái chế
miền ứng dụng bất cứ lúc nào, mà một phép gán cờ bit thì rẻ hơn nhiều so với
việc đi dò một lỗi TLS xuất hiện ngẫu nhiên.

---

## 6. Tách `DichVuThu` ra khỏi `EmailService`

`EmailService` là bản cài `IIdentityMessageService` của ASP.NET Identity. Chữ
ký của giao diện đó — nhận `IdentityMessage`, trả `Task` — chỉ có **một** cách
báo lỗi: **ném ngoại lệ**.

Nhưng nút "Gửi thử" cần biết hỏng **vì lý do gì** để hiện ra cho người đang dò
cấu hình. Một ngoại lệ bay lên trang lỗi chung thì người dùng chỉ thấy *"đã xảy
ra lỗi"* — đúng thứ vô dụng nhất trong tình huống đó.

Nên chia đôi:

```
DichVuThu.GuiAsync(...)  →  KetQuaGuiThu { ThanhCong, ThongBao, ChiTiet }
                            KHÔNG ném ngoại lệ

EmailService.SendAsync() →  lớp vỏ mỏng, gọi DichVuThu rồi NÉM nếu hỏng
                            (Identity mong đợi như vậy: luồng đăng ký
                             phải dừng nếu thư xác minh không gửi được)
```

Cùng một lõi, hai cách báo lỗi cho hai người gọi có nhu cầu khác nhau. Chép
thành hai bản thì bản của Identity sẽ không bao giờ được sửa, vì **không ai
chạy thử nó trực tiếp**.

---

## 7. Diễn giải lỗi sang câu nói rõ phải làm gì

`DichVuThu.DienGiaiLoi` đổi thông báo gốc của .NET thành hướng dẫn cụ thể:

| Triệu chứng | Câu trả về |
|---|---|
| Bắt tay bảo mật hỏng | Máy chủ chỉ chấp nhận TLS 1.2+, hoặc **sai cổng**. Gmail/Office 365 dùng **587**. Cổng **465 (SSL ngầm) KHÔNG dùng được** với `SmtpClient` của .NET |
| `5.7.0` / `5.7.8` / authentication | Với Gmail phải dùng **"Mật khẩu ứng dụng" (App Password) 16 ký tự**, không phải mật khẩu đăng nhập thường — Google bỏ đăng nhập bằng mật khẩu thường từ 2022 |
| No such host / timed out | Sai tên máy chủ hoặc cổng, hoặc tường lửa chặn kết nối ra ngoài |

Hai điều trong bảng này là kiến thức người ta thường mất vài giờ mới tìm ra:

- **Cổng 465 không dùng được.** `System.Net.Mail.SmtpClient` chỉ hỗ trợ
  STARTTLS (cổng 587), **không** hỗ trợ SSL ngầm (cổng 465). Đặt 465 với
  `EnableSsl = true` sẽ treo rồi hết giờ.
- **Gmail cần App Password.** Từ 30/05/2022 Google bỏ "quyền truy cập của ứng
  dụng kém an toàn". Muốn có App Password thì tài khoản Google phải bật xác
  minh 2 bước trước.

---

## 8. Nút "Gửi thử"

**gPortalAdmin → Thiết lập Website → tab Email (SMTP) → mục "Kiểm tra cấu hình"**

Nhập một địa chỉ email, bấm **Gửi thử**. Kết quả hiện trong hộp thoại — thành
công hoặc lý do cụ thể.

### Vì sao cần nút này

Trước đây cách duy nhất để biết SMTP có chạy không là **đăng ký một tài khoản
mới** rồi chờ thư xác minh. Nghĩa là mỗi lần thử cấu hình phải tạo rác trong
bảng người dùng, và khi không nhận được thư thì không biết hỏng ở khâu nào:
cấu hình sai, mật khẩu sai, tường lửa chặn, hay thư rơi vào Spam.

Nút này rút vòng phản hồi từ vài phút xuống vài giây, và quan trọng hơn: nó trả
về **lý do** thay vì im lặng.

### Nó KHÔNG tự lưu cấu hình trước khi gửi

Nghe có vẻ tiện hơn, nhưng "Gửi thử" và "Cập nhật" là **hai ý định khác nhau**.
Trộn lại thì một cú bấm để *thử* sẽ âm thầm *ghi đè* cấu hình đang chạy được
bằng những gì đang gõ dở trên màn hình. Người dùng chỉ định thử, mà mất luôn
cấu hình cũ.

Vì vậy giao diện nói rõ: **bấm Cập nhật trước, rồi mới Gửi thử.**

---

## 9. Mật khẩu SMTP không bao giờ vào nhật ký

`UpdatePortalSettingsEx` có ghi nhật ký (nhóm v), nhưng **chỉ ghi tên các
khóa**, không ghi giá trị:

```csharp
chiTiet: "Các khóa được ghi: " + string.Join(", ", gp_PortalSettingEx.Select(x => x.KeySetting))
```

Danh sách này chứa cả `SMTPPassword`. Nhật ký là thứ **nhiều người đọc được
nhất** trong hệ thống — đổ mật khẩu vào đó là biến bản ghi kiểm toán thành nơi
rò rỉ bí mật.

### Còn hở: mật khẩu SMTP lưu và hiển thị dạng rõ

Nói thẳng để không ai tưởng đây là chỗ đã xong:

- `gp_PortalSettingsEx.ValueSetting` lưu mật khẩu SMTP **dạng chữ rõ**.
- Trong `PortalWebsite.js`, dòng `inputType: 'password'` đang bị **chú thích**,
  nên ô mật khẩu hiện nguyên văn trên màn hình.

Hai điều này chưa nằm trong phạm vi đợt sửa lần này. Cách xử lý đúng là mã hóa
giá trị bằng khóa máy chủ (dự án đã có `gSecurity`), và chỉ trả về chuỗi che
`••••••` cho giao diện — nhưng làm vậy phải xử lý được cả trường hợp "người
dùng để nguyên ô che, tức là không đổi mật khẩu", nên nó là một việc riêng.

---

## 10. Quy trình dò cấu hình SMTP

Chạy theo thứ tự, dừng ở bước đầu tiên hỏng:

**Bước 1 — Cấu hình có thật sự được lưu không?**

```sql
SELECT KeySetting, ValueSetting FROM dbo.gp_PortalSettingsEx
WHERE KeySetting LIKE 'SMTP%';
```

Nếu **không có dòng nào**: bấm Cập nhật lại. Với bản sửa mới, các dòng sẽ được
thêm mới. Nếu vẫn trống thì đọc hộp lỗi hiện ra — nó đã nói lý do.

**Bước 2 — Bấm "Gửi thử".** Đọc kỹ câu trả về, nó đã chỉ ra phải làm gì.

**Bước 3 — Kiểm cấu hình cho từng nhà cung cấp:**

| | Gmail | Office 365 |
|---|---|---|
| Máy chủ | `smtp.gmail.com` | `smtp.office365.com` |
| Cổng | `587` | `587` |
| Tài khoản | địa chỉ Gmail đầy đủ | địa chỉ đầy đủ |
| Mật khẩu | **App Password 16 ký tự** | mật khẩu tài khoản (hoặc App Password nếu bật MFA) |

**Bước 4 — Xem nhật ký hệ thống.** Lọc nhóm **"Lỗi phát sinh"**, tìm hành động
`GUI_THU_KIEM_TRA` hoặc `GUI_THU_THAT_BAI`. Bấm đúp một dòng để xem toàn bộ vết
gọi.

**Bước 5 — Kiểm hộp Spam** trước khi kết luận là không gửi được.

---

## 11. Bước tiếp theo: 2FA qua email

Cấu hình `TwoFactorEnabled` **đã có sẵn** trong `gp_PortalSettings`, và
`AccountController` đã có luồng `VerifyCode` / `SendTwoFactorCodeAsync`
("Email Code"). Nhưng ô cấu hình trong giao diện đang bị **chú thích**, và
luồng này chưa được chạy thử lần nào.

Điều kiện tiên quyết là mục 1–5 ở trên: **2FA qua email chỉ chạy được khi việc
gửi thư đã chạy được.** Bấm "Gửi thử" thành công là điều kiện cần trước khi bật
bất cứ thứ gì liên quan tới 2FA.

---

## 12. Nơi cần biết

| Việc | Tệp |
|---|---|
| Gửi thư + chẩn đoán lỗi | `gPortal.Framework/DichVuThu.cs` |
| Lớp vỏ cho Identity | `gPortal.Framework/Identity/IdentityConfig.cs` (`EmailService`) |
| Bật TLS 1.2 lúc khởi động | `gPortal/Global.asax.cs` (`Application_Start`) |
| Lưu cấu hình mở rộng (đã sửa) | `gPortal.Framework/Store/gp_PortalSettingsExStore.cs` |
| API gửi thử | `AdminController.GuiThuThu` (`POST /Admin/TestEmail`) |
| Giao diện | `PortalWebsite.js` — tab **Email (SMTP)** |

**Không có migration nào** cho đợt sửa này — chỉ là lỗi mã nguồn, cấu trúc CSDL
không đổi.
