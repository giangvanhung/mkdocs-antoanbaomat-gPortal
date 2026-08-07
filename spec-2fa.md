# Đặc tả — Xác thực hai lớp (2FA) qua email

> **Trạng thái:** đặc tả để **tự viết lại**. Code hiện có trong repo là bản do AI
> sinh, dùng làm **tham chiếu đối chiếu**, không phải bản để chép.
>
> Tài liệu này cố ý **không chứa code hoàn chỉnh**. Nó mô tả *cái gì* và *vì sao*,
> để phần *như thế nào* là việc của người viết.

---

## 1. Phạm vi

Bắt người dùng nhập một mã 6 chữ số gửi qua email, sau khi đã nhập đúng mật khẩu,
trước khi được vào hệ thống.

**Ngoài phạm vi:** TOTP / Authenticator app, SMS, khóa phần cứng, mã dự phòng
(backup codes), 2FA cho đường đăng nhập di động.

### Thuật ngữ

| Từ | Nghĩa trong tài liệu này |
|---|---|
| **Mã / OTP** | 6 chữ số, dùng một lần, gửi qua email |
| **Kho mã** | Nơi lưu bản băm của mã đang chờ |
| **Vé nửa chừng** | Cookie chứng nhận "đã qua mật khẩu, chưa qua mã" |
| **Khóa 2FA** | Trạng thái chặn riêng, do nhập sai mã quá nhiều |
| **Cờ toàn hệ thống** | `gp_PortalSettings.TwoFactorEnabled` |

---

## 2. Bản đồ code AI đã sinh

Liệt kê để **biết chính xác cái gì cần thay**, không phải để chép.

### 2.1 File mới hoàn toàn — xóa được nguyên vẹn

| Tệp | Dòng | Vai trò |
|---|---|---|
| `gPortal.Framework/Security/MaXacThucHaiLop.cs` | 187 | Sinh / băm / so khớp mã |
| `gPortal.Framework/Security/XacThucHaiLop.cs` | 551 | Vòng đời mã + chính sách khóa |
| `gPortal.Framework/Security/XacThucHaiLopTokenProvider.cs` | 132 | Nối vào ASP.NET Identity |

### 2.2 Đoạn chèn vào file có sẵn — phải gỡ đúng đoạn

| Tệp | Vị trí | Nội dung |
|---|---|---|
| `Database/DbUpdate.sql` | `519-596` | `[MIGRATION 009]` bảng `gp_TwoFactorOtp` |
| `Database/DbUpdate.sql` | `597-662` | `[MIGRATION 010]` 3 cột trên `gp_Users` |
| `gPortal.Framework/Identity/IdentityModels.cs` | `81`, `97`, `106` | 3 thuộc tính trên `ApplicationUser` |
| `gPortal.Framework/Identity/IdentityModels.cs` | `180` | `IDbSet<gp_TwoFactorOtp>` |
| `gPortal.Framework/Identity/IdentityModels.cs` | `656-756` | `enum MucDichOtp` + `class gp_TwoFactorOtp` |
| `gPortal.Framework/Identity/IdentityConfig.cs` | `110-126` | Ghi đè `GetTwoFactorEnabledAsync` |
| `gPortal.Framework/Identity/IdentityConfig.cs` | `299-307` | Thay `EmailTokenProvider` |
| `gPortal.Framework/gPortal.Framework.csproj` | `134-136` | 3 dòng `<Compile Include>` |
| `gPortal/Controllers/AccountController.cs` | `19` | `using gPortal.Framework.Security;` |
| `gPortal/Controllers/AccountController.cs` | `668-753` | Viết lại `VerifyCode` POST |
| `gPortal/Controllers/AdminController.cs` | `5` | `using Microsoft.AspNet.Identity;` |
| `gPortal/Controllers/AdminController.cs` | `709`, `724-737` | Chặn bật 2FA qua form lưu cấu hình |
| `gPortal/Controllers/AdminController.cs` | `903-1176` | 5 endpoint 2FA |
| `gPortal/Views/Account/VerifyCode.cshtml` | `12-40` | Chú thích + thuộc tính ô nhập |
| `gPortalAdmin/app/view/Website/PortalWebsite.js` | `286-393` | Khối giao diện 2FA |
| `gPortalAdmin/app/view/Website/PortalWebsite.js` | `646`, `809` | Móc vào `init` / `loadSettings` |
| `gPortalAdmin/app/view/Website/PortalWebsite.js` | `893-1076` | 6 handler |

### 2.3 Thứ KHÔNG nên xóa

Hai thứ này là **sửa lỗi có sẵn**, độc lập với 2FA:

- `AdminController.cs:709, 724-737` — bịt đường bật 2FA thẳng từ form lưu cấu hình.
  Không có nó thì quy trình xác minh trước khi bật chỉ là hình thức.
- Bản build `gPortalAdmin` đã triển khai. Sao lưu ở `gPortalAdmin.bak-20260807`.

---

## 3. Phân tầng

Đây là phần đáng đọc kỹ nhất. Bản AI sinh **có phân tách nhưng chưa sạch**;
mục 3.5 nói rõ chỗ hỏng.

### 3.1 Domain — luật thuần, không I/O

Những điều đúng bất kể dữ liệu nằm ở đâu, thư gửi bằng gì:

- Mã gồm **6 chữ số**, phân phối **đều** trên `000000–999999`.
- Mã được lưu dưới dạng **băm có muối riêng từng mã**, không bao giờ lưu bản gốc.
- Phép băm **cố ý chậm**. Không gian mã chỉ `10^6`; hàm băm nhanh khiến việc lưu
  băm gần như vô nghĩa.
- So khớp **hằng thời gian** — thời gian chạy không được tiết lộ sai ở đâu.
- Mã có **hạn dùng 300 giây**, chốt **lúc tạo**, không tính lại lúc kiểm tra.
- Mã gắn với đúng **một cặp `(người dùng, mục đích)`**.
- Mã **dùng được đúng một lần**.
- Sai **5 lần** thì khóa.

Hiện thực: `MaXacThucHaiLop` — thuần, không chạm CSDL. ✅

**Nhưng:** hai hằng số `HanDungGiay = 300` và `NguongSaiToiDa = 5` lại nằm ở
`XacThucHaiLop` (tầng ứng dụng). Đó là **luật miền bị đặt nhầm tầng**.

### 3.2 Application — điều phối, chính sách

Các tình huống sử dụng:

| Use case | Đầu vào | Kết quả |
|---|---|---|
| Phát mã | `userId`, `mucDich` | Mã gốc để gửi đi |
| Gửi mã | `email`, mã gốc | Kết quả gửi thư |
| Đối chiếu mã | `userId`, mã nhập, `mucDich` | `Dung` / `SaiMa` / `HetHan` / `KhongCoMa` |
| Xử lý sai | `userId`, kết quả | Tăng đếm, khóa nếu chạm ngưỡng, cảnh báo |
| Xử lý đúng | `userId` | Đếm về 0 (**không** mở khóa) |
| Hỏi trạng thái khóa | `userId` | `bool` |
| Mở khóa | `userId`, người thực hiện | `bool` |
| Dọn mã hết hạn | — | Số dòng đã xóa |

Hiện thực: `XacThucHaiLop`.

**Nguyên tắc thiết kế phải giữ:** *tách sự thật khỏi chính sách.*
Hàm đối chiếu chỉ trả lời "mã đúng hay sai" và **không** gây hậu quả.
Việc đếm, khóa, gửi cảnh báo nằm ở hàm riêng. Lý do: nhiều nơi cần *biết* mã
đúng/sai mà không được phép làm tăng bộ đếm.

### 3.3 Infrastructure — thế giới bên ngoài

| Thành phần | Hiện thực |
|---|---|
| Kho mã | Bảng `gp_TwoFactorOtp` qua Entity Framework |
| Gửi thư | `DichVuThu` → SMTP |
| Nhật ký | `NhatKyHeThong` → `gp_AuditLogs` |
| Cấu hình | `CacheHelper` + `gp_PortalSettings` |
| Vé nửa chừng | OWIN `TwoFactorCookie`, khai ở `Startup.Auth.cs:65` |
| Bộ nối Identity | `XacThucHaiLopTokenProvider` |
| Bộ nối cờ hệ thống | Ghi đè `GetTwoFactorEnabledAsync` |

### 3.4 API / Presentation

| Điểm vào | Đường dẫn |
|---|---|
| Màn nhập mã | `GET/POST /Account/VerifyCode` |
| Gửi mã để bật | `POST /Admin/TwoFactorSendCode` |
| Xác nhận bật | `POST /Admin/TwoFactorEnable` |
| Tắt | `POST /Admin/TwoFactorDisable` |
| Danh sách bị khóa | `GET /Admin/TwoFactorLockedUsers` |
| Mở khóa | `POST /Admin/TwoFactorUnlock` |

### 3.5 Điểm yếu lớn nhất của bản hiện tại

**Tầng ứng dụng gọi thẳng hạ tầng.** `XacThucHaiLop` tự `new ApplicationDbContext()`,
gọi thẳng `DichVuThu` và `NhatKyHeThong` — cả ba đều là lớp tĩnh hoặc cụ thể.

Hệ quả cụ thể: **không viết được một unit test nào.** Muốn kiểm "mã hết hạn thì
trả về `HetHan`" phải có SQL Server thật, và phải **chờ đủ 5 phút**.

Đó chính là lý do bản hiện tại có **0 test**.

Hướng sửa khi viết lại — đưa bốn thứ này thành phụ thuộc truyền vào:

| Trừu tượng | Vì sao cần |
|---|---|
| Kho mã | Thay bằng bản trong bộ nhớ khi test |
| Bộ gửi thư | Ghi lại thư đã gửi thay vì gửi thật |
| Bộ ghi nhật ký | Kiểm tra đã ghi đúng dòng chưa |
| **Đồng hồ** | **Quan trọng nhất** — `DateTime.UtcNow` rải khắp nơi khiến không test được chuyện hết hạn mà không ngồi chờ thật |

---

## 4. Ba luồng

### 4.1 Đăng nhập

```
Người dùng nhập tên + mật khẩu
        │
        ├─ sai ────────────────────► báo lỗi, tăng bộ đếm mật khẩu
        │
        └─ đúng
             │
             ├─ 2FA tắt ──────────► vào hệ thống
             │
             └─ 2FA bật
                  │
                  ├─ cấp VÉ NỬA CHỪNG (5 phút), chưa cấp quyền gì
                  ├─ phát mã, lưu băm, xóa mã cũ cùng (user, mục đích)
                  ├─ gửi mã qua email
                  └─ chuyển tới màn nhập mã
                         │
                         ▼
              ┌──── Người dùng gõ mã ────┐
              │                          │
              │  1. Lấy userId TỪ VÉ     │  ◄── KHÔNG lấy từ form
              │  2. Kiểm KHÓA 2FA        │  ◄── TRƯỚC khi đối chiếu
              │  3. Đối chiếu mã         │
              │                          │
              └──────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     Đúng             Sai mã           Hết hạn
        │                │                │
   đếm về 0       đếm +1, nếu ≥5      không tăng đếm
   ghi nhật ký    thì khóa + gửi      báo "OTP expired"
   thu vé, cấp    cảnh báo            
   cookie chính   
        │
        ▼
   vào hệ thống
```

**Hai chỗ sai là mất toàn bộ tác dụng:**

1. **Lấy `userId` từ vé, không từ form.** Nhận từ form thì bước mật khẩu biến mất:
   kẻ tấn công gửi `userId` của người khác và chỉ còn phải đoán 6 chữ số. Hai lớp
   thoái hóa thành một lớp. Kèm theo: gửi 5 lần sai là **khóa vĩnh viễn** tài
   khoản bất kỳ, không cần đăng nhập.

2. **Kiểm khóa TRƯỚC khi đối chiếu.** Vì "đúng mã" cố ý **không** mở khóa, thứ duy
   nhất chặn người bị khóa là đúng thứ tự này. Đảo lại thì cơ chế khóa vô hiệu —
   **mà không có lỗi biên dịch, không test nào đỏ**.

### 4.2 Bật 2FA

```
Quản trị bấm "Bật"
   │
   ├─ tài khoản chưa có email ─► từ chối, nói rõ lý do
   │
   ├─ phát mã (mục đích = BẬT, khác mục đích ĐĂNG NHẬP)
   ├─ gửi email
   │      │
   │      ├─ gửi HỎNG ──────► DỪNG. Không bật. Ghi nhật ký.
   │      │                   Không hỏi mã — hỏi mã lúc này là
   │      │                   mời người dùng gõ thứ không tới.
   │      └─ gửi ĐƯỢC
   │             │
   │             ▼
   │      Hỏi mã ──► sai ──► báo lỗi, cho gõ lại (KHÔNG phát mã mới)
   │             │
   │             └─ đúng
   │                  │
   │                  ├─ bật cờ toàn hệ thống
   │                  ├─ CẬP NHẬT BỘ NHỚ TẠM ngay
   │                  └─ ghi nhật ký "User XYZ enabled 2FA"
```

**Vì sao phải nhập mã dù đã đăng nhập:** bước đó **không** để xác minh danh tính —
người bấm đã đăng nhập rồi. Nó chứng minh **đường thư thực sự thông**.

Khi cờ bật, *mọi* lượt đăng nhập phụ thuộc vào SMTP, kể cả lượt kế tiếp của chính
người vừa bấm. SMTP sai → mã không bao giờ tới → **không ai vào được nữa, kể cả để
tắt đi**. Đường sửa duy nhất còn lại là sửa thẳng cơ sở dữ liệu.

So sánh để thấy 2FA khác các cấu hình khác thế nào:

| Đặt sai | Hậu quả |
|---|---|
| Thời gian chờ phiên | Khó chịu, vẫn đăng nhập lại được |
| Số lần nhập sai | Khóa vài người, hết giờ tự mở |
| Dải địa chỉ quản trị | Đã có chốt chống tự khóa |
| **Bật 2FA khi thư hỏng** | **Không có đường về** |

**Vì sao TẮT thì không đòi mã:** nút Tắt là **lối thoát hiểm**. Kịch bản tệ nhất là
2FA đang bật rồi máy chủ thư hỏng. Đòi mã gửi qua thư để tắt nghĩa là đúng lúc thư
hỏng cũng không tắt được — khóa trái lối thoát từ bên trong.

> Nguyên tắc: **thao tác nhằm GỠ phụ thuộc vào một thành phần dễ hỏng thì không
> được phụ thuộc vào chính thành phần đó.**

Đòi nhập lại **mật khẩu** thì hợp lý — mật khẩu không phụ thuộc SMTP. Bản hiện tại
chưa làm; đáng cân nhắc khi viết lại.

### 4.3 Mở khóa

Chỉ quản trị viên. Đặt lại **cả cờ khóa lẫn bộ đếm** — mở khóa mà để bộ đếm ở 5 thì
lần sai kế tiếp khóa lại ngay.

**Không nằm trong đặc tả gốc nhưng bắt buộc phải có:** khóa 2FA không tự hết theo
thời gian như khóa đăng nhập sai, cũng không tự mở khi đổi mật khẩu như khóa do
mật khẩu hết hạn. Thiếu nó thì tài khoản bị khóa là chết vĩnh viễn.

---

## 5. Bất biến — luôn phải đúng

| # | Bất biến | Hỏng thì sao |
|---|---|---|
| B1 | Mã gốc không tồn tại ở đâu ngoài hộp thư người nhận | Lộ CSDL / log là lộ quyền đăng nhập |
| B2 | Mỗi `(user, mục đích)` có tối đa **1** mã sống | Bấm "gửi lại" 10 lần → 10 mã hợp lệ → cơ hội đoán trúng ×10 |
| B3 | Mã đúng → xóa dòng ngay trong cùng lần lưu | Mã dùng rồi vẫn vào được |
| B4 | Hạn dùng chốt lúc tạo | Sửa cấu hình làm sống lại mã đã chết |
| B5 | `userId` lấy từ vé nửa chừng | Xem 4.1 |
| B6 | Kiểm khóa trước khi đối chiếu | Xem 4.1 |
| B7 | Mã đúng **không** mở khóa | Người bị khóa tự cởi bằng cách đoán trúng |
| B8 | Hết hạn **không** tính vào bộ đếm sai | Người đi pha trà quay lại bị khóa |
| B9 | Đối chiếu hỏng → coi như **SAI** (fail-closed) | Lỗi CSDL thành cửa vào |
| B10 | Hỏi trạng thái khóa hỏng → coi như **KHÔNG khóa** (fail-open) | Trục trặc thoáng qua khóa toàn bộ người dùng, kể cả người đi sửa |
| B11 | Bộ đếm OTP tách khỏi bộ đếm mật khẩu | Đăng nhập đúng 1 lần là reset số lần đoán OTP |
| B12 | Cờ 2FA chỉ đổi qua endpoint chuyên dụng | Bật được mà không chứng minh thư thông |
| B13 | Hạn mã = hạn vé nửa chừng | Một bên chết trước → bế tắc |

**B9 và B10 ngược hướng nhau — có chủ ý.** Câu hỏi quyết định không phải "cái nào
an toàn hơn" mà là *"cho qua ở đây thì kẻ tấn công được thêm cái gì?"* Ở B10 họ
được thêm đúng một cơ hội gõ 6 chữ số mà B9 vẫn chặn — tức là không được gì.

---

## 6. Test case

Đánh dấu ★ = bắt buộc, hỏng là hổng bảo mật thật.

### 6.1 Domain — thuần, chạy trong mili-giây

| # | Kiểm |
|---|---|
| D1 | `SinhMa` trả về đúng 6 ký tự, toàn chữ số |
| D2 | Chạy 100.000 lần: có mã bắt đầu bằng `0` (không mất số 0 đầu) |
| D3 | Chạy 100.000 lần: phân phối đều, không giá trị nào lệch bất thường |
| D4 | Hai lần `SinhMuoi` cho hai giá trị khác nhau |
| D5 | Cùng `(mã, muối)` → cùng bản băm |
| D6 ★ | Cùng mã, khác muối → **khác** bản băm |
| D7 | Băm với mã rỗng / muối rỗng → `null`, không ném |
| D8 | Băm với muối không phải Base64 → `null`, không ném, **không** coi là khớp |
| D9 ★ | So khớp: khác 1 ký tự → `false`; khác độ dài → `false`; `null` → `false` |
| D10 | Chuẩn hóa bỏ dấu cách / gạch / chấm; **giữ** chữ cái |

### 6.2 Application — cần kho giả + đồng hồ giả

| # | Kiểm |
|---|---|
| A1 ★ | Phát mã xóa mã cũ **cùng** `(user, mục đích)` |
| A2 ★ | Phát mã **không** xóa mã của mục đích khác |
| A3 ★ | Phát mã **không** xóa mã của người khác |
| A4 | `HetHanUtc == TaoLucUtc + 300s` |
| A5 | Mã đúng → `Dung`, dòng bị xóa |
| A6 ★ | Dùng lại chính mã đó → `KhongCoMa` |
| A7 ★ | Đẩy đồng hồ +301s → `HetHan`, dòng bị xóa |
| A8 | Sai mã → `SaiMa`, `SoLanThu` tăng |
| A9 ★ | Mã của A đem xác minh cho B → không khớp |
| A10 ★ | Mã mục đích BẬT đem dùng cho ĐĂNG NHẬP → không khớp |
| A11 ★ | Kho ném lỗi → trả `SaiMa` (B9) |
| A12 | `HetHan` **không** làm tăng bộ đếm (B8) |
| A13 ★ | Sai lần thứ 5 → khóa, có mốc thời gian, có thư cảnh báo |
| A14 | Sai lần thứ 6 → **không** gửi cảnh báo lần nữa |
| A15 ★ | Đúng mã → đếm về 0, cờ khóa **giữ nguyên** (B7) |
| A16 ★ | Hỏi trạng thái khóa mà kho lỗi → `false` (B10) |
| A17 | Mở khóa → cờ `false` **và** đếm `0` |
| A18 | Dọn rác chỉ xóa mã quá hạn |
| A19 ★ | Không lời gọi nhật ký nào chứa mã gốc (B1) |

### 6.3 API — cần host web

| # | Kiểm |
|---|---|
| P1 | Không có vé nửa chừng → chuyển về đăng nhập |
| P2 ★ | Gửi kèm `userId` trong form → **bị bỏ qua** (B5) |
| P3 ★ | Đang bị khóa + gõ **đúng** mã → vẫn bị chặn (B6) |
| P4 | Mã đúng → vé nửa chừng bị thu hồi, cookie chính được cấp |
| P5 | Mã đúng → bộ đếm sai mật khẩu về 0 |
| P6 | Ba câu lỗi hiện đúng tình huống |
| P7 ★ | Lưu cấu hình với cờ 2FA bật → **không** bật được (B12) |
| P8 ★ | Bật khi SMTP hỏng → không bật, có nhật ký |
| P9 | Bật với mã sai → không bật |
| P10 | Tắt không cần mã |
| P11 | Đủ 3 dòng nhật ký theo đặc tả |
| P12 ★ | Không phản hồi HTTP nào chứa mã gốc |
| P13 | Bật 2FA rồi đăng nhập di động → không lọt |

### 6.4 Thủ công

| # | Kiểm |
|---|---|
| M1 | Chạy `DbUpdate.sql` **hai lần** — không lỗi, không mất dữ liệu |
| M2 | Nhận thư thật, đúng tiêu đề và nội dung |
| M3 | Cột "Bị khóa lúc" hiện đúng giờ Việt Nam |
| M4 | Bật 2FA → đăng xuất → đăng nhập lại được trọn vòng |

---

## 7. Nguồn để tự xác minh

### 7.1 Email làm kênh xác thực lớp hai — đáng đọc trước tiên

**NIST SP 800-63B — Digital Identity Guidelines: Authentication and Lifecycle Management**
<https://pages.nist.gov/800-63-3/sp800-63b.html>

Đọc phần **§5.1.3 Out-of-Band Authenticators**.

Dùng để xác minh nhận định: *email **không** được NIST công nhận là kênh xác thực
ngoài luồng hợp lệ*, vì nhận được thư **không chứng minh sở hữu thiết bị** — hộp
thư chỉ là một thứ khác cũng được bảo vệ bằng mật khẩu.

Điều này liên quan trực tiếp tới thiết kế đang làm: 2FA qua email **yếu hơn** TOTP
hoặc khóa phần cứng. Nó vẫn chặn được kẻ chỉ có mật khẩu bị lộ, nhưng không chặn
được kẻ đã chiếm hộp thư. Đặc tả yêu cầu email, nên ta làm email — nhưng phải biết
rõ giới hạn.

### 7.2 Tổng quan thiết kế MFA

**OWASP — Multifactor Authentication Cheat Sheet**
<https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html>

Đọc phần về các loại yếu tố và điểm yếu của từng loại. Dùng để đối chiếu các quyết
định ở mục 4 và 5.

### 7.3 Cơ chế 2FA sẵn có của ASP.NET Identity

**Mã nguồn ASP.NET Identity 2.x** — <https://github.com/aspnet/AspNetIdentity>

Đọc `TotpSecurityStampBasedTokenProvider.cs` và `EmailTokenProvider.cs`.

> ⚠️ **Phần này tôi chưa xác minh được từ nguồn chính thức.**
> Nhận định "`EmailTokenProvider` suy ra mã từ `SecurityStamp` chứ không lưu, cửa
> sổ khoảng 3 phút và không chỉnh được" là do tôi suy từ hành vi và trí nhớ, **chưa
> mở mã nguồn ra đối chiếu trong phiên làm việc này**.
>
> Đây là nhận định **nền tảng** cho quyết định tự viết kho mã. Nếu nó sai thì phần
> lớn công sức ở mục 3 là thừa. **Nên xác minh trước khi viết dòng code đầu tiên.**

### 7.4 Băm mã có không gian nhỏ

**OWASP — Password Storage Cheat Sheet**
<https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html>

Đọc phần PBKDF2 và số vòng lặp. Lưu ý khi đọc: khuyến nghị ở đó dành cho **mật
khẩu**, nơi độ dài và độ phức tạp gánh phần lớn việc. Mã OTP luôn đúng 6 chữ số —
toàn bộ sức chống đỡ nằm ở **chi phí một lần băm**. Đây là chỗ cần tự cân nhắc chứ
không chép thẳng con số.

---

## 8. Thứ tự viết đề xuất

| Bước | Nội dung | Vì sao trước |
|---|---|---|
| 1 | Xác minh 7.3 | Sai giả định này thì cả thiết kế đổi |
| 2 | Domain + test 6.1 | Thuần, không phụ thuộc gì, chạy được ngay |
| 3 | Trừu tượng hóa kho / thư / nhật ký / đồng hồ | Không có bước này thì 6.2 không viết được |
| 4 | Application + test 6.2 | Nơi chứa gần hết bất biến |
| 5 | Hạ tầng (EF, SMTP) | Việc lắp ráp, ít quyết định |
| 6 | API + test 6.3 | Ba bất biến B5, B6, B12 nằm ở đây |
| 7 | Giao diện | Sửa được liên tục, không khóa quyết định nào |
