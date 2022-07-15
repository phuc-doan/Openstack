## 1. Giới thiệu


- Trong Openstack, Networking Service cung cấp cho người dùng API có thể cài đặt và định nghĩa các network. Gọi là neutron.


- Openstack networking service làm nhiệm vụ tạo và quản lý hạ tầng mạng ảo, bao gồm network, switch, subnet, router


- Openstack networking bao gồm : neutron-server, database storage, plug-in agent, cung cấp một số service khác để giao tiếp với Linux, external devices, or SDN controllers.

## 2. OPS network gồm các thành phần
- **neutron-server** : Chấp nhận và chuyển hướng các API request đến các plugin thích hợp để xử lý.

- **Openstack Networking plug-in và agent** : Có chức năng cắm và gỡ port, tạo mạng hoặc subnet, và cung cấp các địa chỉ IP.

- **Messaging queue** : Được sử dụng để giao tiếp, truyền lệnh và trao đổi thông tin giữa neutron-server với các agent. Nó cũng hoạt động như là một database để lưu trữ trạng thái mạng cho các plug-in cụ thể.

## 3.  Các loại mô hình network trong Openstack Networking 

- Provider,  Routed provider networks và self-service


### 3.1. Provider Network

![image](https://user-images.githubusercontent.com/83824403/178648181-abfbb4b2-09fd-4f24-91f9-9610beead2ab.png)



Provider network cung cấp kết nối mạng layer 2 cho máy ảo với tùy chọn dịch vụ DHCP và metadata. Mạng này kết nối đến mạng layer 2 đã tồn tại ở datacenter, thường dùng Vlan để định danh và chia mạng.

Provider network đơn giản. Chỉ có Admin user mới có quyền chỉnh sửa Provider network vì nó yêu cầu cấu hình hạ tầng mạng vật lý.

Vì L2 nên không hỗ trợ các tính năng như router hay floating ip.



### 3.2. Routed Network 

Routed provider networks cung cấp kết nối ở layer 3 cho các máy ảo. Các network này map với những networks layer 3 đã tồn tại.

Cụ thể hơn , mạng này map tới các các mạng layer 2 đã được chỉ định làm provider network . Mỗi router có một gateway để làm nhiệm vụ định tuyến . . Routed provider networks tất nhiên sẽ có hiệu suất thấp hơn so với provider networks.

### 3.3. Self-service network

![image](https://user-images.githubusercontent.com/83824403/178648137-f8c9ebda-5a64-4886-bfe0-943180640784.png)


- Self-service network cho phép các project quản lý các mạng mà không cần quyền admin. Những mạng này hoàn toàn là mạng ảo và yêu cầu một mạng ảo ,sau đó tương tác với provider network và mạng ngoài ( internet ). Self-service thường sử dụng DHCP và meta cho các instance
- Mong nhiều trường hợp, self-service network sử dụng overload protocol bằng VXLAN, GRE vì chúng có thể hỗ trợ nhiều mạng hơn layer-2 khi sử dụng VLAN ( 802.1q) 
- IPv4 trong self-service thường sử dụng IP Private, tương tác với provider network bằng Source NAT sử dụng router ảo. Floating IP address cho phép truy cập instance từ provider network bằng Destination NAT sử dụng Router ảo 
- IPv6 trong self-service sử dụng IP Public và tương tác với provider network sử dụng các static route thông qua các Router ảo


- User có thể tạo các selt-network theo từng project.  Bởi vậy các kết nối sẽ không được chia sẻ với  các project khác. 

### Trong Openstack hỗ trợ các kiểu cô lập và overlay sau :
- Flat : tất cả instance kết nối vào một network chung. Không được tag VLAN_ID vào packet hoặc phân chia vùng mạng 
- VLAN :  cho phép khởi tạo nhiều provider hoặc tenant network2 sử dụng VLAN (801.2q) , tương ứng với VLAN  đang có sẵn trên mạng vật lý. Điều này cho phép instance kết nối tới các thành phần khác trong mạng
- GRE and VXLAN : là mỗi giao thức đóng gói packet , cho phép tạo ra mạng overlay để tạo kết nối giữa các instance.  Một router ảo để kết nối từ tenant network ra external network.  Một router provider sử dụng để kết nối từ external network vào các instance trong tenant network sử dụng floating IP

<img src="https://github.com/lean15998/Openstack/blob/main/images/07.01.png">
