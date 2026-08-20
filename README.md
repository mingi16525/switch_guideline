# **CẨM NANG TRIỂN KHAI VÀ CẤU HÌNH CORE SWITCH TỔNG THỂ**

**Áp dụng:** Triển khai mới, Nâng cấp, hoặc Chuyển đổi thiết bị Core/Distribution Switch.

**Yêu cầu:** **Cần có bản thiết kế mạng hoặc quy hoạch mạng tổng thể trước khi bắt đầu.**

---

## **GIAI ĐOẠN 1: PHÂN TÍCH THIẾT KẾ VÀ ĐÁNH GIÁ HỆ THỐNG**
**Mục tiêu:** Hiểu rõ hệ thống trước khi gõ bất kỳ dòng lệnh nào.

### **1. Xác định Nền tảng và Hệ điều hành (OS/Hardware)**
*   **Phần cứng:** Số lượng cổng mạng, chuẩn cắm (1G/10G/40G/100G), cáp quang (Singlemode/Multimode), cáp DAC/Breakout.
*   **Hệ điều hành:** **Xác định cú pháp lệnh (Syntax).** Có thể tra tài liệu trên mạng hoặc hỏi AI. Lệnh kiểm tra: `show version`...
*   *Ví dụ:* NX-OS cần bật feature trước khi dùng; Junos dùng kiến trúc edit và commit; Cisco IOS gõ trực tiếp ở global config.
*   **Phần này đặc biệt quan trọng trong các bước chuẩn bị để xác định rõ các lệnh cần sử dụng, cố gắng tối đa để không để xảy ra lỗi câu lệnh trong quá trình cấu hình.**
*   **License/Feature:** Kiểm tra thiết bị có cần license để chạy các giao thức định tuyến L3 (OSPF, BGP) hay không.

### **2. Xác định Vai trò và Kiến trúc Hệ thống**
*   **Kiến trúc Standalone (Single-Core):** Switch chạy độc lập. Gateway đặt trực tiếp trên Interface VLAN (SVI / IRB).
*   **Kiến trúc High Availability (Dual-Core L3):** Chạy 2 Switch Core. Cần giao thức dự phòng Gateway (VRRP, HSRP) và đường liên kết Interconnect.
*   **Kiến trúc Dual-Core L2/L3 (VPC / MC-LAG / VSS):** Gộp 2 switch vật lý thành 1 switch logic để tối ưu chống Loop và cân bằng tải băng thông.

### **3. Phân tích Tài liệu Thiết kế (LLD/HLD)**
*   Trích xuất 3 thông số bắt buộc từ bản thiết kế mạng:
*   **Quy hoạch IP (IP Plan):** Danh sách Subnet, dải IP cấp cho Server/Người dùng, IP Quản trị, và IP Gateway.
*   **Quy hoạch VLAN (VLAN Database):** ID của VLAN, Tên VLAN, Phân loại (Data, Voice, Camera, Management, Storage).
*   **Bản đồ kết nối (Port Mapping):** Cổng nào nối lên Internet/Router, cổng nào nối Server, cổng nào nối Switch Access/PoE.

---

## **GIAI ĐOẠN 2: CHUẨN BỊ VÀ THIẾT LẬP CƠ BẢN (BASELINE)**
**Mục tiêu:** Đưa thiết bị vào trạng thái sẵn sàng quản trị.

### **1. Khôi phục cài đặt gốc / Backup**
*   **Backup cấu hình thiết bị cũ nếu đang làm nhiệm vụ thay thế.**
*   **Lệnh cần sử dụng nghiên cứu tài liệu với hệ điều hành và thiết bị tương ứng.** Lệnh tham khảo: `write erase`...

### **2. Thiết lập định danh**
*   Tên thiết bị (`hostname`).
*   Tài khoản quản trị, phân quyền.

### **3. Truy cập quản trị**
*   Cấu hình IP cho cổng Management (VLAN Quản trị hoặc cổng Mgmt vật lý - Out-of-band).
*   **Bật giao thức SSH, tắt Telnet để bảo mật.**

### **4. Kích hoạt tính năng (Feature Activation)**
*   Bật tính năng Routing (L3).
*   Bật tính năng Link Aggregation (LACP).
*   Bật tính năng khám phá thiết bị (LLDP / CDP).
*   **Các tính năng khác, bổ sung hoàn thiện theo đúng tài liệu thiết kế hoặc phát sinh nếu cần thiết.**

---

## **GIAI ĐOẠN 3: CẤU HÌNH LỚP 2 (LAYER 2 - SWITCHING & VLAN)**
**Mục tiêu:** Tạo môi trường truyền tải và cách ly broadcast domain.

### **1. Khởi tạo VLAN Database**
*   **Khai báo toàn bộ danh sách VLAN theo thiết kế.** Gán tên (Description/Name) rõ ràng để dễ vận hành (VD: `vlan 25 name CAMERA`).

### **2. Quy hoạch các cổng kết nối (Port Configuration)**
*   **Cổng Access (Kết nối thiết bị đầu cuối):** Gán thẳng vào 1 VLAN duy nhất không tag (Untagged).
*   **Cổng Trunk (Kết nối Switch/Router):** Cấu hình cho phép mang nhiều VLAN (Tagged).
*   **Nguyên tắc bảo mật: Luôn dùng lệnh giới hạn VLAN (`allowed vlan`), tuyệt đối không dùng `allow all` để tránh bão broadcast (Broadcast storm).**

### **3. Tối ưu Spanning-Tree**
*   **Với cổng nối trực tiếp vào Server/PC, bật chế độ bỏ qua thời gian hội tụ (Edge Port / Portfast) để link UP ngay lập tức.**
*   Cấu hình Core Switch làm Root Bridge chính (Primary) trong sơ đồ Spanning-Tree.

---

## **GIAI ĐOẠN 4: CẤU HÌNH GOM LINK (LINK AGGREGATION / BONDING)**
**Mục tiêu:** Mở rộng băng thông và tăng khả năng chịu lỗi cáp/cổng.
*Tùy thuộc theo tài liệu thiết kế mạng mà giai đoạn này có thể có hoặc không.*

### **1. Gom link kết nối (Port-Channel / Ether-Channel / LAG)**
*   Nhóm nhiều cổng vật lý thành 1 cổng logic.
*   **Nên sử dụng giao thức đàm phán tự động LACP (Active mode) thay vì ép cứng (Static/On) để hệ thống tự phát hiện lỗi suy hao quang.**

### **2. Cấu hình trên cổng Logic**
*   **Mọi lệnh liên quan đến VLAN, Trunk, Spanning-Tree phải được cấu hình trên Interface Logic (Port-Channel), không cấu hình trên cổng vật lý riêng lẻ.**

---

## **GIAI ĐOẠN 5: CẤU HÌNH LỚP 3 (LAYER 3 - ROUTING & GATEWAY)**
**Mục tiêu:** Giúp các VLAN giao tiếp với nhau và đi ra Internet.

### **1. Cấu hình Default Gateway cho mạng (Inter-VLAN Routing)**
*   **Tạo các giao diện mạng ảo (SVI trên Cisco/Aruba, IRB/RVI trên Juniper).**
*   **Gán IP theo bản quy hoạch (IP Plan). Các IP này sẽ làm Gateway cho thiết bị bên dưới.**
*   **Nếu dùng kiến trúc Dual-Core:** Cấu hình thêm VRRP/HSRP. IP vật lý đặt riêng, cấp phát thêm 1 IP Ảo (VIP) để làm Gateway thực tế.

### **2. Định tuyến**
*   **Default Route (Đường ra Internet): Định tuyến toàn bộ lưu lượng chưa biết (0.0.0.0/0) đẩy ra IP của Router Edge / Firewall.**
*   **Static/Dynamic Route (Tùy chọn):** Nếu có kết nối site-to-site (VPN, Leased Line), cấu hình OSPF, BGP hoặc Static Route cụ thể đến các lớp mạng từ xa.

---

## **GIAI ĐOẠN 6: KIỂM THỬ VÀ NGHIỆM THU (TESTING & VERIFICATION)**
**Mục tiêu:** Đảm bảo mạng chạy đúng logic và không có điểm nghẽn.

### **1. Kiểm tra Layer 1 & 2**
*   **Xác nhận đèn cổng vật lý sáng (Link UP).** Kiểm tra công suất quang (DOM/DDM).
*   **Kiểm tra trạng thái LACP (Bundle UP).**
*   **Xác nhận MAC Address của các thiết bị bên dưới đã được học đúng trên các VLAN.**

### **2. Kiểm tra Layer 3**
*   **Kiểm tra bảng định tuyến (Routing Table) đã học đủ các mạng kết nối trực tiếp và Default Route.**
*   **Ping IP Gateway nội bộ để kiểm tra Inter-VLAN Routing.**
*   **Ping IP Router / Firewall / 8.8.8.8 để xác nhận đường ra Internet thông suốt.**

### **3. Kiểm thử dự phòng (Failover Test - Đối với HA)**
*   **Rút 1 dây cáp trong nhóm LACP, kiểm tra ping có rớt không.**
*   **Khởi động lại (Reboot) 1 Switch Core (nếu chạy Dual), kiểm tra VRRP có tự động đổi trạng thái (Failover) chuyển Gateway sang thiết bị còn lại không.**

---

## **GIAI ĐOẠN 7: BÀN GIAO VÀ ĐƯA VÀO VẬN HÀNH (GO-LIVE)**

### **1. Lưu cấu hình (Save/Commit)**
*   **Đảm bảo cấu hình đang chạy (Running-Config) được lưu vào bộ nhớ khởi động (Startup-Config).**

### **2. Labeling**
*   **Đánh nhãn dây cáp đúng theo bản đồ Port Mapping.**

### **3. Bàn giao tài liệu**
*   **Xuất file cấu hình cuối cùng dưới dạng `.txt`.**
*   **Bàn giao bản vẽ vật lý, thông tin IP/VLAN và mật khẩu quản trị.**
