
## Overview
<a name="2.1"></a>

## 2.1. Một số component tham gia vào quá trình khởi tạo và dự phòng cho máy ảo
- CLI: Command Line Interpreter - là giao diện dòng lệnh để thực hiện các command gửi tới OpenStack Compute
- Dashboard (Horizon): cung cấp giao diện web cho việc quản trị các dịch vụ trong OpenStack
- Compute(Nova): quản lý vòng đời máy ảo, từ lúc khởi tạo cho tới lúc ngừng hoạt động, tiếp nhận yêu cầu tạo máy ảo từ người dùng.
- Network - Quantum (hiện tại là Neutron): cung cấp kết nối mạng cho Compute, cho phép người dùng tạo ra mạng riêng của họ và kết nối các máy ảo vào mạng riêng đó.
- Block Storage (Cinder): Cung cấp khối lưu trữ bền vững cho các máy ảo
- Image(Glance): lưu trữ đĩa ảo trên Image Store
- Identity(Keystone): cung cấp dịch vụ xác thưc và ủy quyền cho toàn bộ các thành phần trong OpenStack.
- Message Queue(RabbitMQ): thực hiện việc giao tiếp giữa các component trong OpenStack như Nova, Neutron, Cinder.

<a name="2.2"></a>
