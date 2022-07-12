## 1.1 Tổng quan nova

- **Nova** là service chịu trách nhiệm chứa và quản lí các hệ thống cloud computing. **OpenStack Nova** là một project core trong OpenStack, nhằm mục đích cấp phép các tài nguyên và quản lý số lượng lớn máy ảo.
- **Nova - OpenStack Compute Service** chính là phần chính quan trọng nhất trong kiến trúc hệ thống **Infrastructure-as-a-Service (IaaS)**. 
- Phần lớn các modules của Nova được viết bằng **Python**.
- **OpenStack Compute** giao tiếp với các service khác của OpenStack :
    - **OpenStack Identity (Keystone)** để xác thực
    - **OpenStack Image (Glance)** để lấy images
    - **OpenStack Dashboard (Horizon)** để lấy giao diện cho người dùng và người quản trị.
    - Ngoài ra còn có thể tương tác với các service khác : block storage, disk, baremetal compute instance
- Nova cho phép điều khiển các máy ảo và cũng có thể quản lí các truy cập tới cloud từ users và projects. 
- Nova không chứa các phần mềm ảo hóa. Thay vào đó, nó sẽ định nghĩa các drivers để tương tác với các kĩ thuật ảo hóa khác chạy trên hệ điều hành của người dùng và cung cấp các chức năng thông qua một web-based API.
## 1.2 Các thành phần của Nova
- **`nova-identify service`**: giao tiếp với keystone để xác thực request
- **`nova-api`** : nhận các yc http sau đó chuyển đổi thành lệnh, thực hiện giao tiếp các thành phần khác nhau qua hàng đợi oslo
- **`nova-compute`** : quản lý vm, khởi tạo máy ảo, gửi state vm vào db .
- **`nova-conductor`** : tương tác nova-compute và DB.
- **`nova-scheduler`** : lấy các yêu cầu máy ảo đặt vào queue và xác định xem chúng được chạy trên compute server host nào.
- **`nova-novncproxy`** : cung cấp VNC proxy thông qua browser, cho phép VNC console để truy cập máy ảo .
- **`nova-volume`**: quản lý vòng đời các ổ dĩa ảo của các instance
- **`DB`**: lưu trạng thái build-time và run-time của cloud infra gồm:

+) Các loại VM có sẵn

+) instance đang SD

+) Netwok có sẵn

+) Project


## 1.3 Quá trình khởi tạo 1 instance 

![image](https://user-images.githubusercontent.com/83824403/178469450-7e38024f-10da-4a76-b472-f8a95d341af8.png)


## Các bước được thực hiện lần lượt như sau:

- **Horizon Dashboard** hoặc **Openstack CLI** lấy thông tin đăng nhập và chứng thực của người dùng cùng với định danh service thông qua REST API để xác thực với **Keystone** sinh ra token.
Sau khi xác thực thành công, client sẽ gửi request khởi chạy máy ảo tới **`nova-api`**.
- **`nova-api`** xác thực lại thông t token với **Keystone** và nhận header với roles và permission
- **`nova-api`** gửi lệnh tới **`nova-conductor`** kiểm tra trong database conflicts hay không để khởi tạo một entry mới.
- **`nova-api`** gửi RPC tới **`nova-scheduler`** service để lập lịch tạo instance
- **`nova-scheduler`** lấy request từ message queue
- **`nova-scheduler`** thông qua filters và weights để tìm compute host phù hợp nhất chạy instance. Đồng thời trên database sẽ cập nhật lại entry của instance với host ID nhận được từ **`nova-scheduler`**. Sau đó **`nova-scheduler`** gửi RPC call tới **`nova-compute`** để khởi tạo máy ảo.
- **`nova-conductor`** lấy request từ message queue.
- **`nova-conductor`** lấy thông tin instance từ database sau đó gửi về cho **`nova-compute`**
- **`nova-compute`** lấy thông tin máy ảo từ queue. Tại thời điểm này, compute host đã biết được image nào sẽ được sử dụng để chạy instance. **`nova-compute`** sẽ hỏi tới **`glance-api`** để lấy url của image .
- **`glance-api`** sẽ xác thực token và gửi lại metadata của image trong đó bao gồm cả url của nó.
- **`nova-compute`** sẽ đưa token tới **`neutron-api`** và hỏi nó về network cho instance.
- Sau khi xác thực token, neutron sẽ tiến hành cấu hình network.
- **`nova-compute`** tương tác với **`cinder-api`** để gán volume vào instance.
- **`nova-compute`** sẽ generate dữ liệu cho Hypervisor và gửi thông tin thông qua libvirt.
