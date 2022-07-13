## 1.1. Khái niệm

- Cinder (Block Storage) được thiết kế với khả năng lưu trữ dữ liệu mà người dùng cuối có thể sử dụng bởi Project Compute (NOVA). Nó có thể được sử dụng thông qua các reference implementation (LVM) hoặc các plugin driver dành cho lưu trữ.
- Có thể hiểu ngắn gọn về Cinder như sau : Cinder là ảo hóa việc quản lý các thiết bị Block Storage và cung cấp cho người dùng một API đáp ứng được như cầu tự phục vụ cũng như yêu cầu tiêu thụ các tài nguyên đó mà không cần có quá nhiều kiến thức về lưu trữ.

## **2) Kiến trúc và cơ chế của Cinder**

<p align=center><img src=https://i.imgur.com/OG5Sx2F.png></p>

- **`cinder-api`** : xác thực và định tuyến các yêu cầu xuyên suốt dịch vụ Cinder
- **`cinder-scheduler`** : Lên lịch và định tuyến các yêu cầu tới dịch vụ volume thích hợp. Tùy thuộc vào cách cấu hình, có thể chỉ là dùng round-robin để định ra việc sẽ dùng volume service nào, hoặc phức tạp hơn có thể sử dụng **Filter Scheduler**. **Filter Scheduler** là mặc định và bật các bộ lọc như **Capacity**(sức chứa), **Avaibility Zone**, **Volume Type**, và **Capability**(khả năng).
- **`cinder-volume`** : Quản lý thiết bị block storage, đặc biệt là các thiết bị back-end
- **`cinder-backup`** : Cung cấp khả năng backup các volume storage sang 1 repo khác
- **`Driver`** : Chứa các mã back-end cụ thể để có thể liên lạc với các loại lưu trữ khác nhau.
- **`Storage`** : Các thiết bị lưu trữ từ các nhà cung cấp khác nhau.
- **`SQL DB`** : Cung cấp một phương tiện dùng để back up dữ liệu từ Swift/Celp, etc,....


## **3) Các thành phần trong Cinder**
- **Back-end Storage Device** : Dịch vụ **Block Storage** yêu cầu một vài kiểu của **back-end storage** mà dịch vụ có thể chạy trên đó. Mặc định là  LVM trên một local volume group tên là "`cinder-volumes`"
- **User** và **Project** : **Cinder** được dùng bởi các người dùng khác nhau (project trong một shared system), sử dụng chỉ định truy cập dưa vào role (role-based access). Các role kiểm soát các hành động mà người dùng được phép thực hiện. Trong cấu hình mặc định, phần lớn các hành động không yêu cầu một role cụ thể, nhưng sysad có thể cấu hình trong file `policy.json` để quản lý các rule. Một truy cập của người dùng có thể bị hạn chế bởi project, nhưng username và pass được gán chỉ định cho mỗi user. Key pairs cho phép truy cập tới một volume được mở cho mỗi user, nhưng hạn ngạch để kiểm soát sự tiêu thu tài nguyên trên các tài nguyên phần cứng có sẵn là cho mỗi project.
- **Volume**, **Snapshot** và **Backup** :
    - Volume : Các tài nguyên block storage được phân phối có thể gán vào máy ảo như một ổ lưu trữ thứ 2 hoặc có thể dùng như là vùng lưu trữ cho root để boot máy ảo. Volume là các thiết bị block storage R/W bền vững thường được dùng để gán vào compute node thông qua iSCSI.
    - Snapshot : Một bản copy trong một thời điểm nhất định của một volume. Snapshot có thể được tạo từ một volume mà mới được dùng gaafnd ây trong trạng thái sẵn sàng. Snapshot có thể được dùng để tạo một volume mới thông qua việc tạo từ snapshot.
    - Backup : Một bản copy lưu trữ của một volume thông thường được lưu ở Swift.


## 4) Cách Cinder hoạt động

![image](https://user-images.githubusercontent.com/83824403/178651983-85ac35a6-9904-444c-b901-cfb1c8d289df.png)


B1: Client gửi YC tạo volume

B2: Cinder API nhận YC => gửi vào queue AMPQ

B3: Cinder volume lấy tin nhắn từ queue gửi cho cinder-scheduler để quyết block storage nào sẽ SD tạo volume

B4: Cinder schedule lấy tin nhắn từ queue, list ra DS backend match tiêu chí storage được chọn

B5: Cinder volume nhận phản hồi từ cinder-schduler trong queue, taọ lần lượt volume trên các backend trong danh sách vừa nhận được cho đến khi success

B6: Cinder volume thu thập metadata và thông tin kết nối volume và gửi đến queue

B7: Cinder API nhận, trả thông tin về volume

B8: Client nhận thông tin trạng thái volume
