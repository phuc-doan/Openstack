# 1. Tổng quan về Keystone

## 1.1. Keystone 

Keystone là một Openstack project cung cấp các dịch vụ Identify, Token, Catalog, Policy cho các dịch vụ khác trong Openstack. Keystone Gồm hai phiên bản:
- v2: sử dụng UUID
- v3: sử dụng PKI, sử dụng một cặp key mở và đóng để xác minh chéo và xác thực. Hai tính năng chính của Keystone:
<ul>
  <ul>
    <li> User Managerment: Keystone giúp xác thực tài khoản người dụng và chỉ định xem người dùng có quyền gì.
    <li> Service Catalog: Cung cấp một danh mục các dịch vụ sẵn sàng cùng với các API endpoint để truy cập các dịch vụ đó.
  </ul>
</ul>

## 1.2 Token

- Quá trình tạo Token(Token Generation Workflow) :

    <img src=https://i.imgur.com/2EgDMz5.png>

    - **B1 :** User request tới Keystone tạo token với các thông tin: `username`, `password`, `project-name`
    - **B2 :** Chứng thực user, lấy `User ID` 
    - **B3 :** Chứng thực project, thu thập thông tin `Project ID` và `Domain ID` từ SQL backend(dịch vụ Resources)
    - **B4 :** Lấy ra role từ backend trên project hoặc domain tương ứng trả về cho user, nếu user không có bất kỳ role nào thì trả về Failure (dịch vụ Assignment)
    - **B5 :** Thu thập các Service và các Endpoint của các service đó (dịch vụ Catalog)
    - **B6 :** Tổng hợp các thông tin về Identity, Resources, Assignment và Catalog ở trên đưa vào token payload, tạo ra token sử dụng hàm `uuid.uuid4().hex`
    - **B7 :** Lưu thông tin của Token vào SQL/KVS backend với các thông tin : `Token ID`, `Expiration`, `Valid`, `UserID`, `Extra`

- Quá trình xác thực Token(Token Validation Workflow) :

    <img src=https://i.imgur.com/3FtVsnO.png>

    - **B1 :** Gửi yêu cầu chứng thực token sử dụng API: `GET v3/auth/tokens` và token (X-Subject-Token, X-Auth-Token)
    - **B2 :** Thu thập token payloads từ token backend KVS/SQL kiểm tra trường `valid`
    - **B3 :** Phân tích token và thu thập metadata: `User ID`, `Project ID`, `Audit ID`, `Token Expire`
    - **B4 :** Kiểm tra token đã expire chưa. Nếu thời điểm hiện tại < "`Token Expire`" theo `UTC` thì token chưa expire,  chuyển sang bước tiếp theo
    - **B5 :** Kiểm tra xem token đã bị thu hồi chưa (kiểm tra trong bảng `revocation_event` của database Keystone)
    - **B6 :** Nếu token đã bị thu hồi (tương ứng với 1 event trong bảng `revocation_event`), trả về thông báo `Token Not Found`. Nếu chưa bị thu hồi, trả về token (truy vấn HTTP thành công `HTTP/1.1 200 OK`)

- **Quá trình thu hồi Token** (***Token Revocation Workflow***) :

    <img src=https://i.imgur.com/8fKCvAi.png>

    - **B1 :** Gửi yêu cầu thu hồi token với API request `DELETE v3/auth/tokens`. Trước khi thực hiện sự kiện thu hồi token thì phải chứng thực token nhờ vào tiến trình Token Validation Workflow đã trình bày ở trên.
    - **B2 :** Kiểm tra trường `Audit ID`. Nếu có, tạo sự kiện thu hồi với `audit id`. Nếu không có `audit id`, tạo sự kiện thu hồi với `token expire`
    - **B3 :** Nếu tạo sự kiện thu hồi token với `audit ID`, các thông tin cần cập nhật vào bảng `revocation_event` của database **Keystone** gồm `audit_id`, `revoke_at`, `issued_before` . Nếu tạo sự kiện thu hồi token với **token expired**, các thông tin cần thiết cập nhật vào bảng `revocation_event` của database **Keystone** bao gồm `user_id`, `project_id`, `revoke_at`,  `issued_before`, `token_expired`
    - **B4 :** Loại bỏ các sự kiện của các token đã expired từ bảng `revocation_event` của database **Keystone**. Cập nhật vào token database, thiết lập lại trường "`valid`" thành `false` (`0`)

- User gửi yc đến keystone tạo token với các username, pass, project name 
- Keystone xác thực, lấy userid từ backend ( LDAP or SQL ) cho việc xác thực và gửi về cho user bộ role ( các quyền hạn của user này ) và đồng thời gửi các request clone đến các service của OPS

## 1.3 Luồng làm việc Keystone 


- 1. User muốn truy cập server vào hệ thống:
**User** gửi thông tin đăng nhập gồm **username/Password** tới **Keystone**
**Keystone** kiểm tra thông tin đăng nhập. Đúng sẽ gửi lại một **Temporary Token** (được sinh ra từ thông tin đăng nhập của User) và **Generic catalog** (danh mục chung) cho User
- 2. User gửi yêu cầu danh sách các project hay service nó có quyền truy cập:
**User** gửi lại **Temporay Token** cùng với yêu cầu danh sách **Project** hay **Service** mà nó được quyền truy cập
**Keystone** sẽ gửi lại danh sách các **Project và Service** mà **User** có quyền truy cập
- 3. Keystone cung cấp danh sách service cho User :
**User** gửi thông tin đăng nhập với **Service** muốn sử dụng.
**Keystone** sẽ gửi lại thông tin **project hay service** (Nếu User có quyền truy cập) kèm với **Token** sử dụng Service đó.
**User** xác định Endpoint chính xác để gọi đến và gửi **request Token** đến Endpoint đó.
- 4. Service sẽ xác minh Token của User :
**Service** sẽ gửi **Token** của User đến **Keystone** để kiểm tra xem **Token** có đúng User không
**Keystone** kiểm tra và gửi lại kết quả xác minh User.
Nếu đúng thì **Service** sẽ gửi request tới **Keystone** xem **User** này có được sử dụng Service hay không.
- 5. Keystone cung cấp thông tin về Token :
**Keystone** sẽ xác nhận (validate) với **Service** là **Token** này đúng của User và khớp với request và xác nhận request đó từ User
**Service** sẽ xác nhận request với **Policy** với User của nó.
**Service** thực hiện yêu cầu của User
**Service** thực hiện request của User (Nếu Policy của User cho phép)
-  6. Service báo lại trạng thái và kết quả cho User :
**Service** sẽ thông báo cho User có thể sử dụng.

## Tóm lại:
- User gửi yc đến keystone tạo token với các username, pass, project name 
- Keystone xác thực, lấy userid từ backend ( LDAP or SQL ) cho việc xác thực và gửi về cho user bộ role ( các quyền hạn của user này ) và đồng thời gửi các request clone đến các service của OPS
