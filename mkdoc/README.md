# Tài liệu các tính năng bảo mật đã bổ sung cho gPortal

Thư mục này giải thích **những gì đã được thêm vào hệ thống, vì sao lại làm như
vậy, và chỗ nào có thể vỡ**. Viết cho người trực tiếp bảo trì mã nguồn, và cho
người phải trình bày lại với cấp trên.

---

## Trả lời trong 60 giây

> **Sếp hỏi: "Em đã làm gì?"**

Năm nhóm tính năng bảo mật. Bốn nhóm đầu xoay quanh một ý: **hệ thống phải tự
đóng cửa khi có dấu hiệu bất thường, chứ không chờ người vận hành phát hiện.**
Nhóm thứ năm trả lời câu hỏi mà bốn nhóm kia không trả lời được: **chuyện gì đã
thực sự xảy ra.**

| # | Tính năng | Câu hỏi nó trả lời |
|---|---|---|
| 1 | Buộc đổi mật khẩu mặc định + đổi định kỳ + hết hiệu lực | "Mật khẩu này do ai đặt, và dùng bao lâu rồi?" |
| 2 | Khóa tài khoản theo số lần sai trong một cửa sổ thời gian | "Có ai đang dò mật khẩu không?" |
| 3 | Thời gian chờ của phiên làm việc | "Máy này còn ai ngồi trước nó không?" |
| 4 | Giới hạn địa chỉ mạng quản trị | "Người này có đang ngồi ở nơi được phép không?" |
| 5 | Nhật ký hệ thống phân theo 5 nhóm | "Đã từng có ai thử chưa?" |

Bốn nhóm đầu đều có **hai nửa**: một màn hình để quản trị viên đặt chính sách,
và một chốt chặn ở tầng thấp để thực thi chính sách đó.

Ngoài ra đã sửa **bốn lỗi khiến SMTP không gửi được thư** — nền bắt buộc cho
xác thực hai lớp qua email. Xem [09](09-smtp-va-gui-thu.md).

---

## Bản đồ tài liệu

| Tệp | Nội dung |
|---|---|
| [00-tong-quan.md](00-tong-quan.md) | Kiến trúc chung. Đọc file này trước. |
| [01-mat-khau-dinh-ky.md](01-mat-khau-dinh-ky.md) | Buộc đổi mật khẩu mặc định (a), đổi định kỳ, hết hiệu lực, thời gian ân hạn |
| [02-khoa-tai-khoan.md](02-khoa-tai-khoan.md) | Khóa tài khoản khi đăng nhập sai nhiều lần |
| [03-thoi-gian-cho-phien.md](03-thoi-gian-cho-phien.md) | Tự đóng phiên khi không có thao tác |
| [04-dia-chi-mang-quan-tri.md](04-dia-chi-mang-quan-tri.md) | Giới hạn địa chỉ mạng được phép quản trị |
| [05-kiem-thu.md](05-kiem-thu.md) | Kịch bản thử từng tính năng |
| [06-van-hanh-va-su-co.md](06-van-hanh-va-su-co.md) | Migration CSDL, đường lùi khi có sự cố |
| [07-cau-hoi-thuong-gap.md](07-cau-hoi-thuong-gap.md) | Những câu hay bị hỏi và cách trả lời |
| [08-nhat-ky-he-thong.md](08-nhat-ky-he-thong.md) | Nhật ký hệ thống 5 nhóm + sắp xếp lại màn hình cấu hình |
| [09-smtp-va-gui-thu.md](09-smtp-va-gui-thu.md) | Vì sao SMTP không gửi được, đã sửa gì, nút Gửi thử |

---

## Cách đọc các khối mã trong tài liệu này

**Mọi khối mã đều có một thanh tiêu đề ghi nó đến từ đâu**, ví dụ:

`gPortal_Portal/gPortal/Authorization/MustChangePasswordGuard.cs:349-382`

- Đường dẫn tính từ **gốc kho mã** `D:\dev\gPortal`.
- Số sau dấu hai chấm là **dòng trong tệp gốc**. Nếu lệch (mã nguồn đã đổi từ
  lúc viết tài liệu), tìm theo **tên hàm** ghi trong đoạn văn đi kèm.
- Ghi `(rút gọn)` nghĩa là đoạn trong tài liệu đã bỏ bớt phần không liên quan —
  mở tệp gốc sẽ thấy dài hơn.
- Ghi `BẢN CŨ` nghĩa là đoạn đó **không còn trong mã nguồn**; nó được giữ lại để
  giải thích lỗi đã sửa.
- Ghi `không phải mã nguồn` là sơ đồ, ví dụ minh họa, hoặc mã giả.
- Ghi `Chạy trong SQL Server Management Studio` / `PowerShell` / `DevTools
  Console` là lệnh để bạn tự chạy, không phải mã của dự án.

---

## Ba nguyên tắc lặp lại trong toàn bộ thiết kế

Nếu chỉ nhớ được ba điều từ tập tài liệu này, hãy nhớ ba điều sau. Chúng giải
thích được phần lớn các quyết định trong mã nguồn.

### 1. Chốt chặn phải nằm ở nơi MỌI request đi qua

Portal này là hệ lai: **24 trang `.aspx` (WebForms)** chạy song song với các
controller MVC. WebForms **không** đi qua `GlobalFilters` của MVC. Đặt rào chắn
bằng action filter thì gõ thẳng `/apps/home.aspx` là lọt.

Vì vậy cả ba chốt chặn đều nằm ở `Global.asax` — tầng `HttpApplication`, nơi
duy nhất mà cả hai đường ống đều phải đi qua.

### 2. Kiểm tra ở giao diện là trải nghiệm, không phải bảo mật

Mọi ràng buộc đều được viết **hai lần**: một lần trong ExtJS để người dùng biết
mình gõ sai ngay lúc gõ, và một lần trong controller vì **gọi thẳng API thì mọi
thứ trong ExtJS đều bị bỏ qua**.

Hai bản đó không phải là trùng lặp thừa: chúng phục vụ hai mục đích khác nhau.
Nhưng chúng có thể lệch nhau, nên chỗ nào lệch được thì phải ghi chú rõ.

### 3. Chọn hướng hỏng dựa trên hậu quả, không dựa trên thói quen

Khi CSDL trục trặc, chốt chặn nên **cho qua** hay **chặn lại**?

Câu trả lời không giống nhau ở mọi chốt, và đây là chỗ dễ sai nhất:

- **Chốt mật khẩu, chốt phiên làm việc → cho qua.** Người đi qua được đó đều
  **đã đăng nhập hợp lệ**. Chặn họ chỉ hạ luôn cả portal, mà lối thoát duy nhất
  (trang đổi mật khẩu) cũng cần chính CSDL đó nên hỏng theo.
- **Chốt địa chỉ mạng → giữ bản quy tắc cũ trong bộ nhớ.** Ở đây "cho qua"
  nghĩa là **cấp thêm quyền truy cập** cho một địa chỉ lẽ ra bị cấm — nặng hơn
  hẳn. Chỉ khi chưa bao giờ đọc được CSDL lần nào thì mới cho qua.

Mọi nhánh `catch` trong mã nguồn đều có một dòng chú thích nói rõ nó chọn hướng
nào **và vì sao**. Đó không phải là ghi chú cho đẹp: một nhánh `catch` im lặng
chính là chỗ mà chốt chặn biến mất mà không ai biết.

---

## Trạng thái hiện tại

| Hạng mục | Trạng thái |
|---|---|
| Biên dịch toàn bộ giải pháp | ✅ Sạch |
| Migration CSDL 001 → 008 | ⚠️ **Phải chạy `Database/DbUpdate.sql` trước khi chạy portal** |
| Chạy thử tính năng 1 (mật khẩu) | ⬜ Chưa |
| Chạy thử tính năng 2 (khóa tài khoản) | ✅ Cửa sổ đếm đã hoạt động |
| Chạy thử tính năng 3 (phiên làm việc) | ⚠️ Đang chẩn đoán — xem [05-kiem-thu.md](05-kiem-thu.md) |
| Chạy thử tính năng 4 (địa chỉ mạng) | ⬜ Chưa |
| Chạy thử tính năng 5 (nhật ký) | ⬜ Chưa |
| Sửa SMTP + nút Gửi thử | ⬜ Chưa chạy thử |
| Build lại `gPortalAdmin` | ✅ Đã build và triển khai |

**Chưa chạy thử thì chưa xong.** Biên dịch sạch chỉ chứng minh mã nguồn hợp lệ
về cú pháp và kiểu dữ liệu — nó không nói gì về việc chốt chặn có thật sự chặn
hay không.
