# 07 — Những câu hay bị hỏi

Mỗi mục gồm câu trả lời ngắn (dùng khi bị hỏi bất chợt) và phần giải thích sâu
hơn nếu người hỏi đào tiếp.

---

## "Sao không dùng thư viện có sẵn cho những cái này?"

**Ngắn:** Có dùng. ASP.NET Identity lo phần khóa tài khoản, OWIN lo phần vé xác
thực. Phần viết thêm là những chỗ thư viện **không có**, hoặc có nhưng hành vi
không khớp với yêu cầu.

**Sâu hơn — ba chỗ cụ thể:**

| Thư viện có | Vì sao vẫn phải viết thêm |
|---|---|
| Identity đếm số lần sai | Nhưng cộng dồn **vô thời hạn** — không có khái niệm "trong khoảng thời gian T" |
| OWIN có `SlidingExpiration` | Nhưng chỉ gia hạn khi đã quá **nửa** vòng đời → cấu hình 15 phút thành "đóng phiên trong khoảng 7–15 phút" |
| Identity có action filter | Nhưng WebForms **không đi qua** GlobalFilters của MVC → 24 trang `.aspx` không được bảo vệ |

---

## "Nếu ai đó gọi thẳng API thì sao?"

**Ngắn:** Mọi ràng buộc đều được kiểm **hai lần** — một lần ở ExtJS cho trải
nghiệm, một lần ở controller là chốt thật. Gọi thẳng API vẫn phải qua chốt thứ
hai.

**Sâu hơn:** Đây là nguyên tắc được áp dụng nhất quán, không có ngoại lệ:

```csharp title="gPortal_Portal/gPortal/Controllers/AdminController.cs:679-685 (rút gọn)"
// AdminController.AddPortalSettings
string loiHanMatKhau = KiemTraCauHinhHanMatKhau(st);
if (loiHanMatKhau != null) { res.code = "100"; res.message = loiHanMatKhau; return Json(res); }
```

Và kiểm **trước cả hai nhánh** thêm/sửa, không phải trong từng nhánh — để không
có đường nào lọt.

Ngoài ra, chốt chặn không nằm ở tầng MVC mà ở `Global.asax`, nên nó phủ cả những
đường mà MVC không biết tới.

---

## "Làm sao chắc chắn nó thật sự chặn?"

**Ngắn:** Có kịch bản kiểm thử cho từng tính năng, và mỗi kịch bản đều có phần
thử **qua API trực tiếp** chứ không chỉ qua giao diện. Xem
[05-kiem-thu.md](05-kiem-thu.md).

**Nói thẳng:** hiện tại tính năng 2 (khóa tài khoản) đã chạy thử và hoạt động.
Ba tính năng còn lại đã biên dịch sạch nhưng **chưa chạy thử đầy đủ**. Biên
dịch sạch chỉ chứng minh mã nguồn hợp lệ về cú pháp; nó không nói gì về việc
chốt chặn có thật sự chặn hay không.

> Đây là câu trả lời trung thực, và nó tốt hơn nhiều so với "em nghĩ là ổn".
> Cái bảo vệ khỏi việc tin nhầm không phải là đọc kỹ code, mà là chạy thật.

---

## "Nếu CSDL hỏng thì chuyện gì xảy ra?"

**Ngắn:** Tùy chốt, và sự khác nhau là cố ý.

| Chốt | Khi CSDL lỗi | Vì sao |
|---|---|---|
| Mật khẩu, phiên làm việc | **Cho qua** | Người đi qua đó **đã đăng nhập hợp lệ**. Chặn chỉ hạ luôn portal, mà lối thoát (trang đổi mật khẩu) cũng cần CSDL nên hỏng theo |
| Địa chỉ mạng | **Giữ bản quy tắc cũ trong bộ nhớ** | Ở đây "cho qua" = **cấp thêm quyền** cho địa chỉ lẽ ra bị cấm |

Mọi trường hợp đều ghi log mức `Error` để sự cố không trôi im lặng.

---

## "Có ảnh hưởng hiệu năng không?"

**Ngắn:** Có, nhưng đã được tính toán để giữ ở mức một câu truy vấn mỗi request,
và không có câu nào cho file tĩnh.

**Chi tiết:**

| Chốt | Chi phí mỗi request |
|---|---|
| Mật khẩu | 1 câu `SELECT`, nhớ trong `HttpContext.Items`; bỏ qua hoàn toàn với `.js/.css/ảnh` |
| Phiên làm việc | **0 câu truy vấn** — mốc nằm trong vé xác thực. Cookie chỉ được phát lại khi mốc đã cũ hơn ~60 giây |
| Địa chỉ mạng | **0 câu truy vấn** trong 60 giây — bộ quy tắc nằm trong bộ nhớ tĩnh |

Chỗ tốn nhất là chốt mật khẩu. Nó buộc phải đọc CSDL chứ không đọc claim trong
cookie — vì claim đóng băng lúc đăng nhập, và người vừa đổi mật khẩu sẽ bị đá
về trang đổi mật khẩu mãi mãi. Xem
[01-mat-khau-dinh-ky.md](01-mat-khau-dinh-ky.md) mục 3.

---

## "Vì sao có tận ba chốt riêng, không gộp thành một?"

**Ngắn:** Vì chúng trả lời ba câu hỏi khác nhau, hỏng theo ba kiểu khác nhau,
và cần ba hướng xử lý lỗi khác nhau.

**Sâu hơn:** Gộp lại thì:

- Không diễn đạt được "có lý do nhưng không chặn" (vùng ân hạn của mật khẩu).
- Không chọn được hướng hỏng riêng cho từng chốt (mục trên).
- Thứ tự ưu tiên biến thành một chuỗi `if` lồng nhau mà không ai dám sửa.

Thứ tự hiện tại — phiên → địa chỉ → mật khẩu — cũng không tùy tiện. Xem
[00-tong-quan.md](00-tong-quan.md) mục 2.

---

## "Chuột và bàn phím thì liên quan gì tới bảo mật?"

**Ngắn:** Nó bịt một lỗ mà đồng hồ máy chủ không thấy được.

**Sâu hơn:** Các ứng dụng ExtJS gọi AJAX liên tục ở nền. Máy chủ đếm theo
request, nên nó thấy phiên "đang hoạt động" **kể cả khi người dùng đã rời khỏi
máy từ lâu**. Một cái tab bị bỏ quên trên máy tính công cộng sẽ sống mãi.

Máy chủ không phân biệt được "đang làm việc" với "tab bị bỏ quên đang tự poll".
Chuột và bàn phím thì phân biệt được.

**Điểm mấu chốt về mặt thiết kế:** đồng hồ client chỉ được phép làm luật **chặt
hơn**, không bao giờ lỏng hơn. Nó không bao giờ tự gọi về máy chủ để gia hạn —
di chuột không tạo ra request nào, đúng nghĩa đen của yêu cầu "không nhận được
yêu cầu từ người dùng". Muốn gia hạn thì phải **bấm nút**, vì một cú bấm nút
chính là một yêu cầu thật.

---

## "Nếu quản trị viên tự khóa mình ra ngoài thì sao?"

**Ngắn:** Có ba lớp bảo vệ, và lớp đầu tiên khiến tình huống đó gần như không
xảy ra được.

1. **Khi lưu:** hệ thống dựng ra bộ quy tắc *sẽ có* sau thao tác, rồi hỏi đúng
   câu mà chốt chặn sẽ hỏi ở request tiếp theo. Không khớp → **từ chối lưu**.
2. **Khi bị chặn:** trang 403 hiện rõ địa chỉ đang dùng, để còn gọi cho người
   vận hành mà đọc đúng con số.
3. **Khi đã lỡ:** một khóa trong `Web.config` tắt toàn bộ chốt chặn.

Điểm đáng nói ở lớp 1: nó **không tự suy luận song song**, mà gọi lại chính hàm
`AdminIpGuard.DuocPhep` mà chốt chặn dùng. Nên câu trả lời "bạn có tự khóa
không" **không thể lệch** với hành vi thật.

---

## "Vì sao không cho nhập tên miền vào danh sách địa chỉ?"

**Ngắn:** Vì tên miền phải tra DNS mới ra địa chỉ, mà DNS thì người khác kiểm
soát và kết quả đổi theo thời gian.

Một chốt chặn phụ thuộc DNS là chốt chặn mà **người khác cầm chìa khóa**. Kẻ
tấn công chiếm được bản ghi DNS là chiếm được quyền quản trị, mà không cần biết
mật khẩu nào cả.

---

## "Tại sao viết chú thích bằng tiếng Việt và dài như vậy?"

**Ngắn:** Vì phần khó nhất của mã bảo mật không phải là *nó làm gì*, mà là *vì
sao nó phải làm như vậy*.

Ví dụ cụ thể: đoạn so khớp đường dẫn dùng `Contains` trông "sai rành rành".
Nhưng nếu đổi thẳng sang `StartsWith`, portal chạy trong thư mục ảo sẽ hỏng —
kể cả trang đổi mật khẩu cũng bị chặn, tạo vòng chuyển hướng vô tận.

Không có chú thích, người sửa tiếp theo sẽ đổi một lỗ hổng lấy một sự cố. Có
chú thích, họ biết phải làm **cả hai việc**: quy chuẩn đường dẫn *rồi mới* so
theo trọn đoạn.

Mỗi nhánh `catch` trong mã nguồn cũng có một dòng giải thích nó chọn hướng nào
và vì sao — vì một nhánh `catch` im lặng chính là chỗ mà chốt chặn biến mất mà
không ai biết.

---

## "Còn thiếu gì không?"

Nói thẳng những chỗ còn hở, tốt hơn là để người khác tìm ra:

| Hạng mục | Trạng thái |
|---|---|
| Chạy thử tính năng 1, 3, 4 | ⬜ Chưa đầy đủ |
| Ứng dụng di động | ⬜ Chưa cập nhật theo ý nghĩa mới của cờ `MustChangePassword` |
| Dữ liệu `gp_PortalSettings` cũ | ⬜ Chưa kiểm xem có bản ghi nào chỉ điền một trong hai ô hạn mật khẩu |
| `WebSecurity.getIpAdress()` đọc `?ip=` từ query string | ⚠️ Vẫn còn — không dùng cho chốt chặn nhưng làm sai địa chỉ trong thông báo/log của bộ lọc SQL injection |
| Xác thực hai lớp (`TwoFactorEnabled`) | ⬜ Có trong cấu hình nhưng chưa nằm trong phạm vi đợt này |
