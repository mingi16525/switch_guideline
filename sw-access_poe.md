# **CẨM NANG TRIỂN KHAI VÀ CẤU HÌNH ACCESS / POE SWITCH TỔNG THỂ**

**Áp dụng:** Triển khai mới, thay thế hoặc mở rộng các Switch Access, Switch PoE tại tầng truy cập mạng (Access Layer).
**Yêu cầu:** **Cần có bản đồ phân bổ port (Port Mapping), quy hoạch VLAN và tính toán tổng công suất PoE dự kiến trước khi bắt đầu.**

---

## **GIAI ĐOẠN 1: PHÂN TÍCH THIẾT KẾ VÀ ĐÁNH GIÁ HỆ THỐNG**
**Mục tiêu:** Nắm bắt quy mô, luồng dữ liệu và khả năng cấp nguồn của thiết bị.

### **1. Xác định Nền tảng và Hệ điều hành (OS/Hardware)**
*   **Phần cứng:** Số lượng cổng đồng (RJ45), cổng quang uplink (SFP/SFP+). Kiểm tra chuẩn cấp nguồn (PoE / PoE+ / PoE++) và **Tổng công suất nguồn của Switch (PoE Power Budget)**.
*   **Hệ điều hành:** **Xác định cú pháp lệnh (Syntax) tùy theo hãng sản xuất (Cisco, Juniper, Aruba, Ruijie...).**
*   **Phần này đặc biệt quan trọng để đánh giá xem switch có đủ công suất nuôi toàn bộ thiết bị đầu cuối (Camera, Speaker, Wifi) hay không. Tránh tình trạng sập nguồn hoặc cháy switch khi cắm full tải.**

### **2. Xác định Vai trò và Kiến trúc Hệ thống**
*   **Kiến trúc tầng Access:** Thiết bị chỉ đóng vai trò chuyển mạch Lớp 2 (Layer 2), cấp nguồn PoE cho thiết bị đầu cuối và đẩy dữ liệu lên Core Switch.
*   **Cơ chế Uplink:** Kết nối lên Core bằng 1 cáp Trunk đơn lẻ hoặc gom cáp (LACP) để tăng băng thông.
*   **Lớp 3 (Routing):** Switch Access KHÔNG làm nhiệm vụ định tuyến (Inter-VLAN routing), Gateway của các thiết bị cắm vào switch này nằm ở Core Switch.

### **3. Phân tích Tài liệu Thiết kế (LLD/HLD)**
*   **Quy hoạch IP Quản trị:** Xác định địa chỉ IP Management cấp riêng cho Switch này và IP Default Gateway (trỏ về Core).
*   **Quy hoạch VLAN:** Nhận diện các VLAN sẽ đi qua Switch này (Ví dụ: VLAN 25 cho Camera, VLAN 23 cho Loa, VLAN 99 để Quản trị).
*   **Bản đồ kết nối (Port Mapping):** Xác định chính xác cổng nào cắm thiết bị đầu cuối, cổng nào làm Uplink.

---

## **GIAI ĐOẠN 2: CHUẨN BỊ VÀ THIẾT LẬP CƠ BẢN (BASELINE)**
**Mục tiêu:** Đưa thiết bị vào trạng thái sẵn sàng quản trị từ xa.

### **1. Khôi phục cài đặt gốc / Backup**
*   **Backup cấu hình thiết bị cũ nếu đang làm nhiệm vụ thay thế.**
*   **Xóa trắng cấu hình hiện tại (factory default) để tránh xung đột cấu hình cũ.** Lệnh tham khảo: `write erase` (Cisco/Nexus), `request system zeroize` (Juniper).

### **2. Thiết lập định danh**
*   Đặt tên thiết bị tuân thủ quy tắc định danh dự án (VD: `SW-CAM-01`, `SW-PHAN-TRAI-02`).
*   Thiết lập tài khoản quản trị (Username/Password) và phân quyền cao nhất.

### **3. Truy cập quản trị (Management Interface)**
*   Khai báo VLAN Quản trị (VD: `vlan 99`).
*   Gán IP tĩnh cho Interface VLAN Quản trị (SVI/IRB) theo thiết kế.
*   **BẮT BUỘC cấu hình Default Gateway (hoặc Default Route) trỏ về IP của Core Switch để IT có thể SSH/Ping thiết bị từ khác mạng.**
*   **Bật giao thức SSH, tắt Telnet để đảm bảo bảo mật.**

### **4. Kích hoạt tính năng (Feature Activation)**
*   Bật giao thức khám phá thiết bị láng giềng (LLDP / CDP). Điều này giúp switch tự động nhận diện thông số nguồn và mạng của Camera/IP Phone.

---

## **GIAI ĐOẠN 3: CẤU HÌNH LỚP 2 (LAYER 2 - SWITCHING & VLAN)**
**Mục tiêu:** Phân chia cổng vật lý vào đúng mạng ảo và tạo luồng kết nối lên Core.

### **1. Khởi tạo VLAN Database**
*   **Khai báo toàn bộ các VLAN có mặt trên switch này.** Đặt tên rõ ràng (VD: `vlan 25 name CAMERA`). *Lưu ý: Không cần tạo các VLAN của Server hay mạng khác nếu chúng không đi qua switch này.*

### **2. Quy hoạch các cổng kết nối (Port Configuration)**
*   **Cổng Access (Cắm Camera, Speaker, PC):** Gán thẳng cổng vào 1 VLAN duy nhất (Untagged). 
*   **Cổng Trunk (Uplink lên Core):** Cấu hình cho phép mang nhiều VLAN (Tagged).
*   **Nguyên tắc bảo mật Uplink: Luôn dùng lệnh giới hạn VLAN (`allowed vlan`), tuyệt đối không dùng `allow all`. Chỉ cho phép các VLAN đang hoạt động trên switch và VLAN Quản trị đi qua.**

### **3. Tối ưu Spanning-Tree**
*   **Với tất cả các cổng Access cắm trực tiếp vào thiết bị đầu cuối, BẮT BUỘC bật chế độ bỏ qua thời gian hội tụ (Edge Port / Portfast). Nếu không, Camera/Loa sẽ mất 30 giây mới nhận được mạng khi khởi động.**
*   Bật tính năng `BPDU Guard` trên các cổng Access (nếu hệ thống yêu cầu) để chống loop mạng khi có người cắm nhầm 1 switch lạ vào mạng.

---

## **GIAI ĐOẠN 4: CẤU HÌNH POE VÀ BẢO MẬT (POE & SECURITY)**
**Mục tiêu:** Đảm bảo nguồn điện ổn định và ngăn chặn các kết nối trái phép.

### **1. Quản lý cấp nguồn PoE**
*   Mặc định các cổng có hỗ trợ PoE sẽ tự động cấp nguồn khi cắm thiết bị (`auto`).
*   **Theo dõi sát sao tổng công suất tiêu thụ (`show power inline`). Tuyệt đối không cắm thêm thiết bị nếu tổng mức tiêu thụ chạm ngưỡng Power Budget của Switch.**

### **2. Bảo mật cổng (Port Security - Tùy chọn theo dự án)**
*   **Bật tính năng giới hạn MAC (MAC Limit / Port Security) trên các cổng Access: Giới hạn tối đa 1-2 địa chỉ MAC trên mỗi cổng để chống loop hoặc ngăn người dùng tự ý cắm thêm hub/switch mini.**

---

## **GIAI ĐOẠN 5: KIỂM THỬ VÀ NGHIỆM THU (TESTING & VERIFICATION)**
**Mục tiêu:** Đảm bảo thiết bị đầu cuối nhận đủ nguồn, mạng và có thể quản trị.

### **1. Kiểm tra Layer 1 (Vật lý & Nguồn PoE)**
*   **Xác nhận đèn tín hiệu (Link UP) trên tất cả các cổng đã cắm cáp.**
*   **Kiểm tra trạng thái PoE (VD: `show power inline`). Xác nhận switch đang cấp nguồn ra thực tế (Watt) và Camera/Speaker đã khởi động thành công.**

### **2. Kiểm tra Layer 2 (Switching)**
*   **Kiểm tra bảng địa chỉ MAC (MAC Address Table): Xác nhận switch đã học được địa chỉ MAC của thiết bị đầu cuối trên đúng VLAN tương ứng.**
*   Kiểm tra cổng Uplink đã chuyển sang trạng thái Forwarding và truyền đúng danh sách VLAN.

### **3. Kiểm tra Quản trị mạng (Management)**
*   **Ping từ Core Switch hoặc máy trạm IT xuống IP Quản trị của Switch Access để xác nhận kết nối quản lý thông suốt.**
*   Dùng lệnh `show lldp neighbors` để xem switch có nhận diện được thông tin chi tiết của Camera/Speaker cắm vào không.

---

## **GIAI ĐOẠN 6: BÀN GIAO VÀ ĐƯA VÀO VẬN HÀNH (GO-LIVE)**

### **1. Lưu cấu hình (Save/Commit)**
*   **Tuyệt đối không quên lưu cấu hình đang chạy (Running-Config) vào bộ nhớ khởi động (Startup-Config).** Nếu quên, thiết bị sẽ mất toàn bộ cấu hình khi mất điện.

### **2. Labeling (Đánh nhãn)**
*   **Đánh nhãn cáp Uplink và các cáp thiết bị đầu cuối cắm vào switch tuân thủ đúng Port Mapping.**

### **3. Bàn giao tài liệu**
*   **Xuất file cấu hình cuối cùng dưới dạng `.txt` (show running-config).**
*   **Đặc biệt: Phải ghi nhận lại bảng công suất PoE thực tế (Total PoE power consumed) tại thời điểm cắm full tải vào biên bản bàn giao để làm cơ sở bảo trì sau này.**
