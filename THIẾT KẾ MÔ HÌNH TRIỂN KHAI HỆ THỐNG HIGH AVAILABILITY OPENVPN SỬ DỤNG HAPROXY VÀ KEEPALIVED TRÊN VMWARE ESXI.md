THIẾT KẾ MÔ HÌNH TRIỂN KHAI HỆ THỐNG HIGH AVAILABILITY OPENVPN SỬ DỤNG HAPROXY VÀ KEEPALIVED TRÊN VMWARE ESXI
----
## 1. Mục tiêu triển khai

Mục tiêu của giải pháp là xây dựng một hệ thống VPN có tính sẵn sàng cao (High Availability) trên nền tảng VMware ESXi, đáp ứng các yêu cầu sau:

- Người dùng chỉ sử dụng duy nhất một địa chỉ VPN để kết nối từ Internet.
- Hệ thống tự động chuyển đổi sang máy chủ VPN dự phòng khi máy chủ chính gặp sự cố.
- Hệ thống sử dụng HAProxy làm điểm truy cập duy nhất (Single Entry Point).
- Keepalived đảm nhiệm cơ chế VRRP nhằm loại bỏ điểm lỗi đơn (Single Point of Failure) của HAProxy.
- Hạ tầng dễ mở rộng khi bổ sung thêm nhiều máy chủ VPN hoặc triển khai VMware vCenter, vMotion, HA Cluster trong tương lai.

## 2. Kiến trúc triển khai

Do mỗi máy chủ Dell R440 chỉ được trang bị hai cổng mạng vật lý (vmnic0 và vmnic1), phương án triển khai được lựa chọn là sử dụng hai uplink vật lý kết hợp với VMware vSwitch và Port Group để tạo các mạng logic (VLAN) bên trong ESXi.

Trong thiết kế này:

- vmnic0 được sử dụng cho toàn bộ lưu lượng WAN.
- vmnic1 được sử dụng cho lưu lượng nội bộ (Transit, VPN và LAN).
- Việc phân tách các miền mạng được thực hiện bằng Port Group và VLAN trên VMware ESXi.

Giải pháp này giúp tối ưu băng thông, giảm Hairpin Traffic và phù hợp với mô hình triển khai trong môi trường doanh nghiệp.

## 3. Mô hình kết nối tổng thể

<img width="605" height="730" alt="image" src="https://github.com/user-attachments/assets/f2e10578-dc69-4fb6-906a-5e98d2514e0e" />

## 4. Thiết kế hạ tầng mạng
### 4.1 Phân chia cổng mạng vật lý

**vmnic0**

Được sử dụng cho kết nối WAN.

Chức năng:

- Nhận lưu lượng từ FortiGate.
- Tiếp nhận kết nối VPN từ Internet.
- Trao đổi gói tin VRRP giữa hai HAProxy.
- Mang Virtual IP của Keepalived.

**vmnic1**

Được sử dụng cho toàn bộ lưu lượng nội bộ.

Bao gồm:

- Kết nối giữa HAProxy và OpenVPN.
- Lưu lượng VPN sau khi xác thực.
- Lưu lượng đi đến Cisco L3 Switch.
- Kết nối đến pfSense.
- Truy cập vào mạng LAN.

Việc tách riêng WAN và LAN giúp giảm tải cho uplink và hạn chế hiện tượng Hairpin Traffic.

## 5. Thiết kế VMware ESXi

Trên mỗi máy chủ ESXi tạo hai Virtual Switch.

**vSwitch0**

Uplink:

vmnic0

Port Group

| Port Group | VLAN  | Chức năng |
|------------|-------|-----------|
| WAN        | VLAN10 | Kết nối FortiGate |
| HA         | VLAN11 | VRRP Keepalived (có thể dùng chung với WAN nếu phù hợp thiết kế) |

**vSwitch1**

Uplink:

vmnic1

Port Group

| Port Group | VLAN | Chức năng |
|------------|:----:|-----------|
| VPN-Transit | VLAN20 | HAProxy ↔ OpenVPN |
| LAN | VLAN30 | OpenVPN ↔ Cisco L3 ↔ pfSense |
| Management | VLAN40 | Quản trị ESXi (khuyến nghị) |

Thiết kế này cho phép mở rộng thêm các VLAN như Backup, vMotion hoặc Storage mà không cần bổ sung card mạng vật lý.

## 6. Thiết kế địa chỉ IP
**Mạng WAN**

| Thiết bị | Địa chỉ IP |
|----------|------------|
| Gateway FortiGate | 6.0.1.1 |
| HAProxy1 | 6.0.1.2 |
| HAProxy2 | 6.0.1.3 |
| Virtual IP (Keepalived VIP) | 6.0.1.10 |

**Mạng VPN**

| Thiết bị | Địa chỉ IP |
|----------|------------|
| Gateway | 6.0.2.1 |
| OpenVPN1 | 6.0.2.2 |
| OpenVPN2 | 6.0.2.3 |

## 7. Vai trò của từng thành phần
### FortiGate
- Firewall Internet.
- NAT Public IP về Virtual IP của HAProxy.
- Chỉ cho phép các cổng VPN (ví dụ TCP/443).

### HAProxy

HAProxy là điểm truy cập duy nhất của toàn bộ hệ thống.

Chức năng:

- Tiếp nhận kết nối từ Internet.
- Kiểm tra trạng thái OpenVPN.
- Chuyển tiếp kết nối đến máy chủ VPN khả dụng.
- Hỗ trợ cân bằng tải hoặc Active/Standby.

### Keepalived

Keepalived triển khai giao thức VRRP.

Chức năng:

- Quản lý Virtual IP.
- Tự động chuyển VIP giữa HAProxy1 và HAProxy2.
- Loại bỏ Single Point of Failure của HAProxy.

### OpenVPN

Máy chủ OpenVPN chịu trách nhiệm:

- Xác thực người dùng.
- Thiết lập VPN Tunnel.
- Chuyển lưu lượng vào mạng nội bộ.

Hai máy chủ có cấu hình giống nhau nhằm đảm bảo khả năng thay thế lẫn nhau khi xảy ra sự cố.

### Cisco Layer 3 Switch

Chức năng:

- Kết nối hai máy chủ ESXi với pfSense.
- Thực hiện định tuyến giữa các VLAN (nếu cần).
- Mở rộng hạ tầng mạng.

### pfSense

Đóng vai trò Firewall nội bộ.

Bao gồm:

- NAT.
- Firewall Policy.
- Routing.
- VPN Routing.
- Quản lý lưu lượng vào hệ thống.

## 8. Luồng hoạt động của hệ thống

### Bước 1

Người dùng truy cập:

vpn.company.vn

Tên miền được phân giải về Public IP của FortiGate.

### Bước 2

FortiGate thực hiện NAT:

Public IP

↓

6.0.1.10 (Virtual IP)

### Bước 3

Keepalived xác định máy HAProxy đang giữ Virtual IP.

Ví dụ:

HAProxy1

đang là MASTER.

Toàn bộ lưu lượng được chuyển đến HAProxy1.

## Bước 4

HAProxy thực hiện Health Check đối với:

OpenVPN1
OpenVPN2

Nếu OpenVPN1 hoạt động:

Client

↓

HAProxy1

↓

OpenVPN1

Nếu OpenVPN1 không hoạt động:

Client

↓

HAProxy1

↓

OpenVPN2

## Bước 5

OpenVPN xác thực người dùng và tạo Tunnel VPN.

## Bước 6

Lưu lượng VPN được chuyển qua vmnic1 đến Cisco Layer 3 Switch.

## Bước 7

Cisco Layer 3 Switch chuyển tiếp lưu lượng đến pfSense.

## Bước 8

pfSense định tuyến và áp dụng chính sách Firewall trước khi cho phép truy cập vào mạng nội bộ.

## 9. Kịch bản High Availability
### Trường hợp 1

HAProxy1 hoạt động bình thường.

Keepalived giữ Virtual IP trên HAProxy1.

Toàn bộ người dùng kết nối đến HAProxy1.

### Trường hợp 2

HAProxy1 gặp sự cố.

Keepalived tự động chuyển Virtual IP sang HAProxy2.

Người dùng vẫn sử dụng cùng một địa chỉ VPN.

Không cần thay đổi cấu hình Client.

### Trường hợp 3

OpenVPN1 gặp sự cố.

HAProxy loại bỏ OpenVPN1 khỏi Backend.

Các kết nối mới tự động chuyển sang OpenVPN2.

### Trường hợp 4

Dell R440 số 1 gặp sự cố.

HAProxy2 trở thành MASTER.

OpenVPN2 tiếp nhận toàn bộ kết nối VPN.

Hệ thống vẫn duy trì dịch vụ với thời gian gián đoạn tối thiểu.

## 10. Lộ trình triển khai

Việc triển khai được thực hiện theo trình tự sau:

Cấu hình VMware ESXi.
Tạo vSwitch và Port Group.
Cấu hình VLAN trên Cisco L3 Switch.
Triển khai Ubuntu Server.
Cài đặt OpenVPN.
Cài đặt Easy-RSA và xây dựng hệ thống CA.
Tạo chứng chỉ Server và Client.
Cài đặt HAProxy.
Cấu hình HAProxy Backend.
Cài đặt Keepalived.
Cấu hình VRRP và Virtual IP.
Cấu hình NAT trên FortiGate.
Cấu hình Routing trên pfSense.
Kiểm thử các kịch bản Failover.
Giám sát hệ thống và tối ưu hiệu năng.

## 11. Kết luận

Phương án sử dụng hai uplink vật lý kết hợp với VMware vSwitch và VLAN là giải pháp tối ưu đối với hạ tầng hiện tại gồm hai máy chủ Dell R440 chỉ có hai cổng mạng vật lý.

Thiết kế này đáp ứng đồng thời các yêu cầu về tính sẵn sàng cao (High Availability), khả năng mở rộng và hiệu năng mạng. Hệ thống tránh được hiện tượng Hairpin Traffic, phân tách rõ ràng lưu lượng WAN và LAN, đồng thời tạo nền tảng thuận lợi để mở rộng lên các công nghệ như VMware vCenter, vMotion, pfSense HA hoặc SD-WAN trong tương lai mà không cần thay đổi kiến trúc mạng cơ bản.
