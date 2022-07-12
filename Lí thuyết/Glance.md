# Glance

## 1.1 Tổng quan về Glance
- Glance là Image Service của Openstack bao gồm việc tìm kiếm, thu thập, cung cấp image cho các máy ảo.
- Glance cung cấp RESTful API cho phép quản lý, truy vấn image, metadata.
- Các volume snapshot có thể sử dụng để backup hoặc làm template cho máy ảo mới.
- Glance được thiết kế để có thể là dịch vụ hoạt động độc lập cho sự cần thiết các tổ chức lớn lưu trữ và quản lý các disk image ảo.
- Image của glance có thể được lưu ở nhiều vị trí khác nhau như file system thông thường hay hệ thống lưu trữ object như Swift.

## 1.2 Luồng hoạt động của Glance 

- Quá trình tạo images
B1: Tạo image, image lúc này đuọc đưa vào queue và được nhận diện trong thời gian ngắn, sẵn sàng tải lên => chuyển sang trạng thái queue
- Chuyển sang trạng thái `saving` nghĩa là quá trình tải lên chưa hoàn thành
B2: 
- Khi image được tải lên xong, trạng thái image chuyển sang `active`
- Nếu quá trình tải thất bại thì nó chuyển sang trạng thái `killed` hoặc `deleted`
- Có thể `deactive` (tắt) hoặc `reactive` (bật) các image đã upload thành công bằng command.

<p align=center><img src="https://i.imgur.com/OERBwfx.png"></p>

## 1.3 Các trạng thái của image


- `queued` : trạng thái của image được bảo vệ trong **glance registry**. Không có dữ liệu nào của image được tải lên **Glance** và kích thước của image không được thiết lập về zero khi khởi tạo.
- `saving` : biểu thị rằng dữ liệu của image đang được upload lên **glance**. Khi một image đăng ký với một call đến POST /image và có một x-image-meta-location header, image đó sẽ không bao giờ được trong tình trạng `saving` (dữ liệu Image đã có sẵn ở vị trí khác).
- `active` : biểu thị một image đó là hoàn toàn sẵn sàng trong **Glance**. Điều này xảy ra khi các dữ liệu image được tải lên.
- `deactivated` : Trạng thái biểu thị việc không được phép truy cập vào dữ liệu của image với tài khoản không phải admin. Khi image ở trạng thái này, ta không thể tải xuống cũng như export hay clone image.
- `killed` : trạng thái biểu thị rằng có vấn đề xảy ra trong quá trình tải dữ liệu của image lên và image đó không thể đọc được
- `deleted` : trạng thái này biểu thị việc **Glance** vẫn giữ thông tin về image nhưng nó không còn sẵn sàng để sử dụng nữa. Image ở trạng thái này sẽ tự động bị gỡ bỏ vào ngày hôm sau.
