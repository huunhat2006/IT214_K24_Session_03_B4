# Bài 4: Phân tích sự cố Service Discovery khi scale hệ thống

## 1. Phân tích lỗi cấu hình Eureka Client
### Lỗi 1: `spring.application.name` không nhất quán
- **Hiện trạng:** Tên ứng dụng được đặt là `RestaurantService` (viết hoa CamelCase), trong khi các service khác lại dùng kebab-case chữ thường (ví dụ: `order-service`).
- **Phân tích rủi ro:** Mặc dù Eureka Client vẫn có thể đăng ký lên Eureka Server thành công, nhưng Eureka quản lý ID của các service trong Registry của nó bằng cách tự động chuyển đổi tên ứng dụng thành UPPERCASE (viết hoa toàn bộ, ví dụ `RESTAURANTSERVICE`). Tuy nhiên, ở phía client (`order-service`), khi lập trình viên dùng RestTemplate, FeignClient hoặc cấu hình route ở Gateway, họ thường giả định tên service tuân theo quy ước kebab-case chữ thường (chuẩn phổ biến của Spring) là `restaurant-service`. Sự bất đồng bộ trong cách gọi tên này có thể khiến bộ phân giải địa chỉ (Load Balancer) không thể map URI được yêu cầu với tên service thực tế có trên Eureka, dẫn đến lỗi `UnknownHostException` hoặc `Service Unavailable` (503).
- **Cách khắc phục:** Luôn tuân thủ quy ước kebab-case chữ thường để nhất quán: `spring.application.name: restaurant-service`.

### Lỗi 2: Thiếu dấu `/` ở cuối `defaultZone`
- **Hiện trạng:** Cấu hình trỏ URL là `defaultZone: http://eureka-server:8761/eureka`
- **Phân tích rủi ro:** Eureka Client ở phía dưới (under-the-hood) sử dụng Jersey REST client để gọi API giao tiếp với Eureka Server. Jersey yêu cầu cấu hình root path của REST API phải luôn kết thúc bằng dấu gạch chéo `/`. Nếu URL bị thiếu, các thao tác gọi HTTP (như POST để đăng ký hoặc PUT để gửi heartbeat) có thể tạo ra đường dẫn sai định dạng, dẫn tới lỗi trả về HTTP 404 từ máy chủ, khiến việc đăng ký lên Eureka Server thất bại hoàn toàn.
- **Cách khắc phục:** Thêm dấu `/` vào cuối: `http://eureka-server:8761/eureka/`.

## 2. Phân tích cơ chế Heartbeat của Eureka
- **Cơ chế hoạt động chung:** Sau khi đăng ký thành công, mỗi Eureka Client sẽ liên tục gửi một tín hiệu HTTP (gọi là Heartbeat) đến Eureka Server định kỳ (mặc định là mỗi 30 giây) để báo cáo rằng "tôi vẫn đang sống".
- **Kịch bản khi `restaurant-service-2` bị crash đột ngột:**
  1. Khi bị crash (sập nguồn, lỗi hệ điều hành) mà không gọi thủ tục tắt grace shutdown, service này không kịp gửi tín hiệu `CANCEL` (tín hiệu thông báo ngắt kết nối chủ động) cho Eureka.
  2. Eureka Server lúc này sẽ không nhận được heartbeat từ instance này nữa.
  3. Tuy nhiên, Eureka Server không lập tức xóa bỏ (evict) instance đó khỏi danh sách để tránh tình trạng mạng bị chập chờn tạm thời. Theo cấu hình mặc định (`eureka.instance.lease-expiration-duration-in-seconds = 90`), Eureka Server sẽ chờ trong khoảng thời gian là 90 giây (tương đương miss 3 nhịp heartbeat).
  4. Hơn nữa, tiến trình dọn dẹp các instance quá hạn (Eviction Task) của Eureka chỉ chạy 60 giây một lần.
  - **Kết luận:** Sẽ mất **từ 90 giây đến tối đa 150 giây** (nếu không kích hoạt tính năng tự bảo vệ Self-Preservation) kể từ khi crash để Eureka Server chính thức loại bỏ `restaurant-service-2` khỏi Registry. Trong khoảng thời gian "chết lâm sàng" này, `order-service` vẫn có thể nhận danh sách cũ chứa `restaurant-service-2` và gọi vào nó, gây ra lỗi timeout tạm thời (cần cấu hình retry/circuit breaker để bọc lỗi).

## 3. Đề xuất cấu hình cho `order-service`
Để `order-service` luôn tự động tra cứu và điều hướng (Load Balancing) đến các instance còn sống (trong số 4 instance) của `restaurant-service` thay vì gọi vào một địa chỉ IP fix cứng, ta cần:

1. **Bật chế độ Fetch Registry:** Đảm bảo `eureka.client.fetch-registry: true` (mặc định) trong `application.yml` của `order-service` để nó liên tục tải (pull) danh sách cập nhật từ Eureka Server về lưu vào cache cục bộ định kỳ.
2. **Sử dụng Load Balancer (Client-side Load Balancing):** 
   - Sử dụng annotation `@LoadBalanced` trên Bean `RestTemplate` của ứng dụng.
   - Khi đó, thay vì gọi URL với IP cứng (`http://192.168.1.10:8085/api/...`), code chỉ cần gọi qua tên dịch vụ ảo: `http://restaurant-service/api/...`. Spring Cloud LoadBalancer sẽ tự tra cứu tên "restaurant-service" trong bộ cache Registry và phân phối request luân phiên (round-robin) đến 4 instance.
3. **Thay thế bằng Spring Cloud OpenFeign (Khuyến nghị):**
   - Thay vì dùng RestTemplate thủ công, định nghĩa một Interface `@FeignClient(name = "restaurant-service")`. FeignClient đã tích hợp sẵn cơ chế Load Balancer cục bộ để tự động load balance giữa 4 instance và hỗ trợ cấu hình gọi lại (retry) nếu vô tình gọi trúng instance vừa crash.
