# NCT Cinema Backend

Spring Boot 4.1.2 / Spring Security / JPA / MySQL. Backend phục vụ frontend Next.js tại thư mục `../nct-cinema-frontend`.

## Chạy trên máy hiện tại

Mở Terminal tại thư mục backend:

```powershell
.\mvnw.cmd spring-boot:run
```

Backend: http://localhost:8080/api/health

Mở Terminal khác:

```powershell
cd "..\nct-cinema-frontend"
npm.cmd run dev
```

Giao diện: http://localhost:3000. Quản trị: http://localhost:3000/admin.

MySQL cần chạy và có database `cinema_booking`. Nếu chưa có:

```sql
CREATE DATABASE cinema_booking CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Ứng dụng chỉ tạo/quản lý các bảng bắt đầu bằng `nct_`. Không nhập hoặc ghi đè bảng của ứng dụng khác. Tạo schema qua JPA `ddl-auto=update` dành cho môi trường phát triển; khi triển khai thật cần quy trình migration và sao lưu.

## Cấu hình riêng tư

File `.env` nằm ở gốc backend, đã được thêm vào `.gitignore`. Lần triển khai này đã chuyển cấu hình DB hiện có vào đó và tạo mật khẩu quản trị ngẫu nhiên, không đưa mật khẩu vào mã nguồn.

- Tài khoản quản trị: xem `NCT_ADMIN_EMAIL` và `NCT_ADMIN_PASSWORD` trong `.env`.
- Đăng ký trên giao diện chỉ tạo tài khoản CUSTOMER.
- Bootstrap chỉ tạo admin khi email chưa tồn tại; đổi biến mật khẩu sau đó không tự đổi mật khẩu tài khoản đã lưu.
- Đặt `NCT_SEED=false` nếu không muốn tạo danh mục phim/rạp mẫu. Seed chỉ chạy khi các danh mục còn trống, không ghi đè dữ liệu đã có.
- Các giá trị trong biến môi trường hệ điều hành có thể ghi đè file cấu hình.

Các biến hỗ trợ:

| Biến | Ý nghĩa |
| --- | --- |
| `DB_URL` | JDBC URL, mặc định MySQL `localhost:3306/cinema_booking` |
| `DB_USER`, `DB_PASSWORD` | Tài khoản database |
| `NCT_ADMIN_EMAIL`, `NCT_ADMIN_PASSWORD` | Tạo admin lần đầu, mật khẩu tối thiểu 12 ký tự |
| `NCT_SEED` | Tạo dữ liệu mẫu nếu danh mục trống |
| `VNPAY_TMN_CODE`, `VNPAY_HASH_SECRET` | Thông tin merchant VNPAY sandbox |
| `VNPAY_RETURN_URL` | Mặc định `http://localhost:3000/payment/return` |
| `PAYMENT_PROVIDER` | Mặc định `vnpay` |
| `MOCK_PAYMENTS_ENABLED` | Mặc định false; mô phỏng chỉ khi đồng thời provider=mock |
| `SERVER_ADDRESS`, `PORT` | Mặc định `127.0.0.1`, `8080` |

Có thể tham khảo `.env.example`. Không đưa `.env` lên Git hoặc chia sẻ khóa VNPAY.

## VNPAY sandbox

Điền hai khóa merchant trong `.env`, sau đó khởi động lại backend:

```properties
VNPAY_TMN_CODE=ma_website_cua_ban
VNPAY_HASH_SECRET=khoa_bi_mat_cua_ban
VNPAY_RETURN_URL=http://localhost:3000/payment/return
```

Cấu hình **IPN URL** trong tài khoản merchant trỏ tới:

```text
https://dia-chi-cong-khai-cua-ban/api/payments/vnpay/ipn
```

VNPAY không gọi được localhost. Khi thử thanh toán đầy đủ, cần địa chỉ HTTPS công khai trỏ đến frontend/proxy hoặc backend và cấu hình IPN tương ứng. Ứng dụng hiện chỉ chạy cục bộ; không tự mở tunnel hay triển khai công khai. Return URL cần cùng hostname với trang bạn dùng để đăng nhập (ví dụ dùng `localhost` xuyên suốt, hoặc đổi toàn bộ sang `127.0.0.1`).

Luồng:

1. Đăng nhập, chọn suất và ghế.
2. `POST /api/tickets` tạo vé pending và giữ ghế tối đa 5 phút hoặc đến lúc suất chiếu bắt đầu.
3. `POST /api/tickets/{id}/payment/vnpay` tạo URL có chữ ký HMAC-SHA512; số tiền lấy từ DB, không lấy từ trình duyệt.
4. VNPAY gọi IPN. Backend kiểm tra chữ ký, merchant, mã vé, số tiền và kết quả trước khi xác nhận vé. IPN gửi lại không cộng tiền/giữ ghế lần hai.
5. Trang return chỉ đọc trạng thái vé từ backend, không tin `vnp_ResponseCode` trên trình duyệt để đánh dấu đã thanh toán.

Nếu thiếu khóa merchant, hệ thống báo cổng thanh toán chưa cấu hình; không tự giả lập thành công. Thông báo thanh toán đến sau khi vé đã hết hạn/hủy không được lấy lại ghế đã trả; trạng thái `requires_review` cần rạp kiểm tra và xử lý với nhà cung cấp.

Vé pending có thể hủy. Vé VNPAY đã thanh toán không tự hủy/hoàn tiền qua API hiện tại; cần rạp xử lý hoàn tiền riêng. Không có API hoàn tiền giả cho giao dịch VNPAY.

Tài liệu giao thức: [VNPAY thanh toán](https://sandbox.vnpayment.vn/apis/docs/thanh-toan-pay/pay.html), [mô hình kết nối](https://sandbox.vnpayment.vn/apis/docs/mo-hinh-ket-noi/).

## API

Tất cả thay đổi dùng session cookie và CSRF. Lấy `GET /api/auth/csrf`, gửi giá trị `token` vào header có tên `headerName`. Sau đăng nhập, lấy token mới cho các thay đổi tiếp theo. Frontend tự thực hiện qua proxy `/api`.

| Nhóm | API |
| --- | --- |
| Tài khoản | `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` |
| Danh mục công khai | `GET /api/catalog`, `/api/movies?query=&status=`, `/api/movies/{id}`, `/api/showtimes?movieId=&cinemaId=&date=` |
| Ghế | `GET /api/showtimes/{id}/seats` |
| Vé của tài khoản | `POST /api/tickets`, `GET /api/tickets`, `GET /api/tickets/{id}`, `POST /api/tickets/{id}/cancel` |
| Thanh toán | `GET /api/payments/config`, `GET /api/tickets/{id}/payment`, `POST /api/tickets/{id}/payment/vnpay`, `GET /api/payments/vnpay/ipn` |
| Quản trị | `GET /api/admin/catalog`; POST/PUT/DELETE các nhóm `/api/admin/movies`, `/cinemas`, `/rooms`, `/showtimes`; PUT/DELETE dùng `/{id}` |
| Khóa/mở ghế | `PATCH /api/admin/rooms/{id}/seats/{label}` với `{ "enabled": false }` hoặc true |
| Vé/quản trị | `GET /api/admin/tickets`, `POST /api/admin/tickets/{id}/cancel` |
| Báo cáo | `GET /api/admin/revenue?from=YYYY-MM-DD&to=YYYY-MM-DD` |

Đặt vé:

```json
{
  "showtimeId": "id_suat_chieu",
  "seats": ["D5", "D6"],
  "idempotencyKey": "uuid-cho-moi-lan-dat"
}
```

Gửi lại cùng khóa và cùng lựa chọn trả về cùng vé; cùng khóa với lựa chọn khác bị từ chối. Giá và trạng thái thanh toán từ client bị bỏ qua. Customer chỉ đọc/hủy vé của mình. Backend khóa phòng trong giao dịch và có ràng buộc duy nhất suất/ghế, kiểm tra trùng lịch, ghế khóa và lịch sử vé trước khi sửa/xóa dữ liệu.

Báo cáo dùng ngày đặt vé tại Việt Nam và chỉ tính vé confirmed có payment paid; loại trừ pending, expired, cancelled. Đơn vị tiền là VND nguyên.

## Chạy không cần MySQL

```powershell
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local"
```

Profile local dùng H2 lưu file trong `data/`, không dùng database MySQL. Cấu hình merchant và admin vẫn lấy từ `.env`. Mô phỏng chỉ dành cho thử nghiệm được bật rõ ràng bằng `PAYMENT_PROVIDER=mock` và `MOCK_PAYMENTS_ENABLED=true`.

## Kiểm tra / đóng gói

```powershell
.\mvnw.cmd test
.\mvnw.cmd package
java -jar target\Cinema-0.0.1-SNAPSHOT.jar
```

Kiểm thử dùng profile test và H2 tách biệt; không sửa database MySQL. Kiểm tra session/CSRF/quyền, giá server, vé của tài khoản, tranh chấp ghế đồng thời, giữ ghế hết hạn, hủy/đặt lại, dữ liệu đã có vé, trùng lịch và chữ ký/IPN VNPAY. Có vector chữ ký đối chiếu độc lập.

Java tối thiểu 17; môi trường hiện tại Java 26. Khi đưa lên môi trường công khai cần HTTPS, cookie Secure, cấu hình DB/merchant riêng và quản lý admin phù hợp. Không có mật khẩu admin công khai mặc định.


T?c v? n?n ki?m tra v? h?t h?n m?i gi?y v? tr? gh? trong giao d?ch c? kh?a ph?ng. API xem gh?/??t v? c?ng ki?m tra h?t h?n ngay khi ???c g?i, n?n kh?ch ti?p theo kh?ng ph?i ch? m?t l??t t?c v? n?n. V? confirmed/paid ???c gi? nguy?n. Th? l?i c?ng y?u c?u kh?ng gia h?n th?i gian gi? gh?.
#   N C T _ C i n e m a _ F r o n t e n d  
 