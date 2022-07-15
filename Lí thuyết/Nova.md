## 1.1 Tổng quan nova

- **Nova** là service chịu trách nhiệm chứa và quản lí các hệ thống cloud computing. **OpenStack Nova** là một project core trong OpenStack, nhằm mục đích cấp phép các tài nguyên và quản lý máy ảo.
- **Nova - OpenStack Compute Service** chính là phần chính quan trọng nhất trong kiến trúc hệ thống **Infrastructure-as-a-Service (IaaS)**. 
- Các modules của Nova được viết bằng **Python**.
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
- **`DB`**: lưu trạng thái build-time và run-time của cloud gồm ( Available instance types, Instances in use, Available networks Projects)



## 1.3 Quá trình khởi tạo 1 instance 




![image](https://user-images.githubusercontent.com/83824403/178639606-e6886114-6dcb-4af5-81a2-2ca090da4474.png)






## Các bước được thực hiện lần lượt như sau
- Bước 1: Từ Dashboard hoặc CLI, nhập thông tin chứng thực (ví dụ: user name và password) và thực hiện lời gọi REST tới Keystone để xác thực
- Bước 2: Keystone xác thực thông tin người dùng và tạo ra một token xác thực gửi trở lại cho người dùng, mục đích là để xác thực trong các bản tin request tới các dịch vụ khác thông qua REST
- Bước 3: Dashboard hoặc CLI sẽ chuyển yêu cầu tạo máy ảo mới thông qua thao tác "launch instance" trên openstack dashboard hoặc "nova-boot" trên CLI, các thao tác này thực hiện REST API request và gửi yêu cầu tới nova-api
- Bước 4: nova-api nhận yêu cầu và hỏi lại keystone xem auth-token mang theo yêu cầu tạo máy ảo của người dùng có hợp lệ không và nếu có thì hỏi quyền hạn truy cập của người dùng đó.
- Bước 5: Keystone xác nhận token và update lại trong header xác thực với roles và quyền hạn truy cập dịch vụ lại cho nova-api
- Bước 6: nova-api tương tác với nova-database
- Bước 7: Dababase tạo ra entry lưu thông tin máy ảo mới
- Bước 8: nova-api gửi rpc.call request tới nova-scheduler để cập cập entry của máy ảo mới với giá trị host ID (ID của máy compute mà máy ảo sẽ được triển khai trên đó). C(Chú ý: yêu cầu này lưu trong hàng đợi của Message Broker - RabbitMQ)
- Bước 9: nova-scheduler lấy yêu cầu từ hàng đợi
- Bước 10: nova-scheduler tương tác với nova-database để tìm host compute phù hợp thông qua việc sàng lọc theo cấu hình và yêu cầu cấu hình của máy ảo
- Bước 11: nova-database cập nhật lại entry của máy ảo mới với host ID phù hợp sau khi lọc.
- Bước 12: nova-scheduler gửi rpc.cast request tới nova-compute, mang theo yêu cầu tạo máy ảo mới với host phù hợp.
- Bước 13: nova-compute lấy yêu cầu từ hàng đợi.
- Bước 14: nova-compute gửi rpc.call request tới nova-conductor để lấy thông tin như host ID và flavor(thông tin về RAM, CPU, disk) (chú ý, nova-compute lấy các thông tin này từ database thông qua nova-conductor vì lý do bảo mật, tránh trường hợp nova-compute mang theo yêu cầu bất hợp lệ tới instance entry trong database)
- Bước 15: nova-conductor lấy yêu cầu từ hàng đợi
- Bước 16: nova-conductor tương tác với nova-database
- Bước 17: nova-database trả lại thông tin của máy ảo mới cho nova-conductor, nova condutor gửi thông tin máy ảo vào hàng đợi.
- Bước 18: nova-compute lấy thông tin máy ảo từ hàng đợi
- Bước 19: nova-compute thực hiện lời gọi REST bằng việc gửi token xác thực tới glance-api để lấy Image URI với Image ID và upload image từ image storage.
- Bước 20: glance-api xác thực auth-token với keystone
- Bước 21: nova-compute lấy metadata của image(image type, size, etc.)
- Bước 22: nova-compute thực hiện REST-call mang theo auth-token tới Network API để xin cấp phát IP và cấu hình mạng cho máy ảo
- Bước 23: quantum-server (neutron server) xác thực auth-token với keystone
- Bước 24: nova-compute lấy thông tin về network
- Bước 25: nova-compute thực hiện Rest call mang theo auth-token tới Volume API để yêu cầu volumes gắn vào máy ảo
- Bước 26: cinder-api xác thực auth-token với keystone
- Bước 27: nova-compute lấy thông tin block storage cấp cho máy ảo
- Bước 28: nova-compute tạo ra dữ liệu cho hypervisor driver và thực thi yêu cầu tạo máy ảo trên Hypervisor (thông qua libvirt hoặc api - các thư viện tương tác với hypervisor)






## 1.4: Chọn host khi launch instance

- Compute sử dụng nova-scheduler service để xác định request được gửi tới node compute nào

VD: Đang ở vùng khả dụng được yêu cầu hay không, đủ ram, disk, có thể phục vụ yc, đáp ứng các thuộc tính,... hay không

- Khi filter schedulẻ nhân được yc cho tài nguyên, nó sẽ lọc xem node nào đủ điều kiện đê phân phối 
-  Lúc này list ra được 1 list server đủ ĐK ( tiêp tục nó sẽ dựa vào `weight` để quyết định ) node compute thích hợp
