## 1. Giới thiệu


- Trong Openstack, Networking Service cung cấp cho người dùng API có thể cài đặt và định nghĩa các network. Gọi là neutron.


- Openstack networking service làm nhiệm vụ tạo và quản lý hạ tầng mạng ảo, bao gồm network, switch, subnet, router


- Openstack networking bao gồm : neutron-server, database storage, plug-in agent, cung cấp một số service khác để giao tiếp với Linux, external devices, or SDN controllers.

## 2. OPS network gồm các thành phần
- **neutron-server** : Chấp nhận và chuyển hướng các API request đến các plugin thích hợp để xử lý.

- **Openstack Networking plug-in và agent** : Có chức năng cắm và gỡ port, tạo mạng hoặc subnet, và cung cấp các địa chỉ IP.

- **Messaging queue** : Được sử dụng để giao tiếp, truyền lệnh và trao đổi thông tin giữa neutron-server với các agent. Nó cũng hoạt động như là một database để lưu trữ trạng thái mạng cho các plug-in cụ thể.

## 3.  Các loại mô hình network trong Openstack Networking 

## Tìm hiểu network traffig flow của Neutron khi sử dụng Linux Bridge.

Linux bridge mechanism driver chỉ sử dụng Linux bridge và **veth** để kết nối. Một layer 2 quản lý Linux bridge trên mỗi compute node và node bất kỳ nào khác mà cung cấp Layer3 (routing),  DHCP, Metadata, hoặc các dịch vụ mạng khác.

## 3.1. Trong mô hình mạng Provider.
### Một số ví dụ về mô hình mạng Provider sử dụng Linux bridge
- Trong mô hình mạng Provider dưới đây có hai node với yêu cầu như sau:
  - Controller node: 
    - Có hai network interface: management và provider.
    - Cài đặt Openstack Networking server và Ml2 Plugin.
  - Compute node:
    - Có hai network interface: management và provider. 
    - Cài đặt Linux bridge agent, DHCP agent, metadata agent, và các phụ thuộc khác.

![](https://i.imgur.com/UGowcLN.png)

- Các thành phần trong Openstack networking sẽ sử dụng mạng management để giao tiếp với nhau. Trong khi đó, mạng provider được sử dụng để cung cấp mạng cho máy ảo thông quan Linux Bridge agent và đồng thời cung cấp mạng cho DHCP và metadata agent để cung cấp DHCP và Metadata cho máy ảo.


- Mô hình dưới đây mô tả kết nối trên một compute node với mạng untagged(flat). Thường thì DHCP agent và Metadata agent sẽ nằm trên compute node.

![](https://i.imgur.com/8A0giSh.png)

- Mô hình dưới đây mô tả kết nối giữa các thành phần của Neutron trên Compute node với 2 mạng được tag vlan.

![](https://i.imgur.com/5A5t2xj.png)

### Luồng đi của lưu lượng trong mạng

Phần sau sẽ mô tả luồng đi của lưu lượng mạng trong hai kịch bản phổ biến là North-south và East-west. 
Lưu lượng mạng North-south di chuyển giữa máy ảo và mạng bên ngoài như là internet.
Lưu lượng mạng East-west di chuyển giữa các máy ảo trong cùng hoặc khác mạng với nhau. 
Ở tất cả các kịch bản, hạ tầng mạng vật lý phụ trách switching và routing giữa mạng provider với mạng ngoài như Internet.


#### Kịch bản North-south: Máy ảo với địa chỉ IP cố định.


![](https://i.imgur.com/cPqRZx0.png)

Các bước sau mô tả luồng đi của gói tin được gửi từ máy ảo ra Internet:
1. Interface trên máy ảo(1) chuyển gói tin đến port của instance trên bridge(2) thông qua *veth pair*.
2. Security group rules(3) trên Provider bridge xử lý tường lửa và theo dõi kết nối cho gói tin.
3. VLAN sub-interface(4) trên provider bridge chuyển tiếp gói tin đến interface vật lý(5).
4. Interface mạng vật lý(5) thêm VLAN tag cho gói tin và gửi nó đi đến switch ở hạ tầng mạng vật lý(6)
5. Switch nhận gói tin, gỡ bỏ vlan tag và chuyển tiếp đến router(7)
6. Router định tuyến gói tin từ mạng provider(8) đến mạng ngoài(9).
7. Mạng ngoài nhận gói tin.

#### Kịch bản East-west 1: Giữa các máy ảo trong cùng mạng.

![](https://i.imgur.com/D7XNhdB.png)

Các bước sau mô tả luồng đi của gói tin được gửi từ một máy ảo đến máy ảo khác trong cùng provider network nhưng trên hai compute node khác nhau:
1. Interface trên máy ảo(1) chuyển gói tin đến port của instance trên bridge(2) thông qua *veth pair*.
2. (3)-(4) là linux bridge dùng các thuật toán xử lý và đẩy gói tin đến int vật lý (5).
3. Interface vật lý(5) thêm vlan tag và chuyển tiếp gói tin đến provider switch(6) trong hạ tâng mạng vật lý.
4. Switch chuyển tiếp gói tin từ compute node 1 đến compute node 2.
5. Interface mạng vật lý của compute 2(8) gỡ vlan tag trên gói tin và chuyển tiế nó đến vlan sub-interface port(9) trên provider bridge.
6. Security group rule(10) trên provider bridge xử lý tường lửa và theo dõi kết nối cho gói tin.
7. port của máy ảo trên provider bridge(11) nhận và chuyển tiếp gói tin đến interface của máy ảo(12) thông qua *veth pair* .




#### Kịch bản East-West 2: Hai máy ảo khác mạng.

Trong kịch bản này sẽ giải thích các bước trong luồng đi của gói tin gửi từ hai máy ảo trong cùng một compute node nhưng khác Vlan(VLAN 101 và VLAN 102).

![](https://i.imgur.com/JGd73P3.png)

Các bước:
1. Interface của máy ảo 1(1) chuyển tiếp gói tin đến port của máy ảo(2) trên Provider bridge thông qua **veth pair**.
2. (3)-(4) là linux bridge dùng các thuật toán xử lý và đẩy gói tin đến int vật lý (5).
3. Interface vật lý(5) thêm vlan 101 tag và chuyển tiếp gói tin đến provider switch(6) trong hạ tâng mạng vật lý.
4. Switch  gỡ vlan 101 tag trên gói tin và chuyển cho router(7).
5. Router định tuyến gói tin từ mạng provider 1(8) đến mạng provider 2(9).
6. Router chuyển hướng gói tin đến switch(10).
7. Switch gắn vlan 102 tag cho gói tin và chuyển gói tin đến compute 1(11).
8. Interface vật lý trên compute 1 (12) gỡ vlan 102 tag và chuyển cho Vlan sub-interface port trên provider bridge của mạng provider 2.
9. Security group rule(14) trên provider bridge xử lý tường lửa và theo dõi kết nối cho gói tin.
10. Port của máy ảo trên provider bridge(15) nhận và chuyển tiếp gói tin đến interface của máy ảo 2 (16) thông qua *veth pair* .


### 3.2 Trong mô hình mạng Self-service
### Ví dụ về mô hình mạng Self-service sử dụng 
- Ví dụ về kiến trúc mạng sefl service:

![](https://imgur.com/4WceQla.png)

- Ví dụ dưới đây hiển thị các thành phần và kết nối giữa một mạng self-service và một mạng provider với router đứng ở giữa để thực hiện SNAT.

![](https://i.imgur.com/s08dciE.png)


### Luồng đi của lưu lượng mạng.
Với mô hình mạng self-service, chúng ta cũng sẽ mô tả luồng đi của lưu lượng mạng theo một số kịch bản phổ biến như North-south và East-west.
Lưu lượng mạng North-south di chuyển giữa máy ảo và mạng bên ngoài như là internet.
Lưu lượng mạng East-west di chuyển giữa các máy ảo trong cùng hoặc khác mạng với nhau. 
Ở tất cả các kịch bản, hạ tầng mạng vật lý phụ trách switching và routing giữa mạng provider với mạng ngoài như Internet.


#### Kịch bản North-south 1: máy ảo với địa chỉ ip được fix cứng.

Kịch bản này, máy ảo sẽ gửi gói tin từ mạng self-service 1: 

![](https://i.imgur.com/q3wlcuB.png)

Các bước thực hiện trên compute node:
1. Interface của máy ảo(1) gửi gói tin đến  port của máy ảo(2) trên provider bridge thông qua **veth pair**
2. Security group rule(3) sẽ xử lý firewall và theo dõi kết nối 
3. Self-service bridge chuyển gói tin đến VXLAN interface(4) để đóng gói gói tin sử dụng VNI 101.
4. Interface vật lý(5) được dùng bởi VXLAN interface chuyển tiếp gói tin mà đã được đóng gói qua mạng overlay(6).

Các bước sau thực hiện trên Network node:
1. Interface vật lý(7) được dùng bởi VXLAN interface trên network node chuyển tiếp gói tin đến VXLAN interface(8) để gỡ lớp đóng gói VNI.
2. Port của router trên self-service bridge(9) chuyển tiếp gói tin đến interface mạng self-service(10) trên Router namespace. 
  - Đối với địa chỉ IPv4, router sẽ thực hiện SNAT trên gói tin, nó sẽ thay đổi địa chỉ IP nguồn của gói tin thành địa chỉ IP của router trên mạng provider, và gửi nó đến *gateway IP* của mạng Provider thông qua *gateway interface* của router(11).
  - Đối với địa chỉ IPv6, router sẽ gửi gói tin đến địa chỉ IP next-hop, thường là địa chỉ gateway của mạng provider.
3. Router chuyển tiếp gói tin đến port trên provider bridge(12).
4. Port VLAN sub-interface(13) trên provider bridge chuyển tiếp gói tin đến interface vật lý của mạng provider(14).
5. Interface vật lý(14) gán VLAN tag cho gói tin và chuyển tiếp gói tin ra hạ tầng mạng vật lý(15).

> Luồng đi của gói tin trả về giống với các bước trên nhưng ngược lại. Tuy nhiên, không có địa chỉ floating IPv4, máy chủ ở mạng provider hoặc external network không thể khởi tạo kết nối đến máy ảo ở trong mạng self-service.

#### Kịch bản North-south 2: Máy ảo có địa chỉ floating IPv4.

Với máy ảo có địa chỉ Floating IP, network node sẽ thực hiện SNAT với lưu lượng north-south gửi từ máy ảo ra mạng provider và thực hiện DNAT với lưu lượng north-south gửi từ mạng provider vào máy ảo. Với IPv6 thì không cần phải có Floating Ip và NAT vì trong cả hai trường hợp trên thì network node định tuyến IPv6. Còn muốn biết thêm tại sao thì tìm hiểu thêm về IPv6 đi :))).

Với luồng đi của gói tin từ máy ảo đi ra thì giống với kịch bản đầu tiên nhưng thay vì thay vì sửa Ip nguồn của gói tin thành địa chỉ Ip provider của router thì SNAT sẽ thay địa chỉ IP nguồn thành Floating IP.

Ở đây chúng ta thực hiện kịch bản máy ở mạng provider gửi gói tin đến máy ảo trong mạng self-service.

![](https://i.imgur.com/nfMLBHc.png)

Trên network node :
1. Từ mạng vật lý (1) gửi packet vào provider physical interface(2) trên network node.  
2. Interface vật lý mạng provider(3) sẽ bỏ các VLAN_TAG và sẽ gửi các packet vào VLAN Sub_interface tương ứng  trên provider bridge(4). 
3. Provider bridge sẽ chuyển packet sang self-service router gateway port (5) 
  - Với IPv4, router đảm nhiệm thực hiện DNAT để thay đổi địa chỉ đích IP thành  địa chỉ IP của instance trên mạng self-service.(6) 
4. Router chuyển gói tin đến tin đến port của router trên self-service bridge(7)
5. Self-service bridge gửi packet đến VXLAN Interface kèm theo đóng gói gói tin vói VNI(8)
6. Interface vật lý(9) dùng cho VXLAN  gửi packet đến compute node thông qua Overlay network ( 10 )

Trên compute node:
1. Physical interface (11) dùng cho VXLAN trên compute node nhận gói tin và gửi đến VXLAN interface ở self-server bridge (12) 
2. Security group đảm nhiệm filter packet (13)
3. Port của máy ảo trên self-service bridge(14) chuyển packet đến port của máy ảo(15) thông qua **veth pair**.
 
#### Kịch bản East-west 1: Cùng mạng.
Các máy ảo với địa chỉ IPv4/IPv6 cứng hoặc floating IPv4 trên cùng mạng sẽ giao tiếp trực tiếp giữa các compute node chứa máy ảo. 

Giao thức VXLAN mặc định thiếu thông tin về vị trí đích và dùng multicast để khám phá nó. Sau khi khám phá được, nó lưu vị trí đích trong database nội bộ. Tuy nhiên, với hệ thống lớn, quá trình khám phá này có thể gây ra lưu lượng mạng lớn trên hệ thống mà tất cả các node cần xử lý. Để tăng tính hiệu quả, dịch vụ Openstack Networking sử dụng layer-2 population mechanism driver để tự động quảng bá database cho các VXLAN interface.

Thực hiện kịch bản hai hai máy ảo cùng một mạng self-service, và trên hai compute node khác nhau. 

![](https://i.imgur.com/DaLh5eC.png)

Trên Compute 1:
1. Interface của máy ảo(1) chuyển packet đến port của máy ảo trên self-service bridge(2)
2. Security group (3) xử lý vấn đề liên quan đến tường lửa và theo dõi kết nối. 
3. Self-service bridge chuyển tiếp packet tới VXLAN interface để đóng gói gói tin với VNI.
4. Interface vật lý dùng cho VXLAN(5) trên Compute node 1 chuyển tiếp packet tới compute 2 nhờ overlay network ( 6 ) 

Trên Compute 2 :
1. Trên interface vật lý xử dụng cho VXLAN(7) trên compute node 2 nhận và chuyển tiếp gói tin đến tới VXLAN interface(8) trên self-service bridge. 
2. Security group ( 9 ) xử lý vấn đề liên quan đến tường lửa và theo dõi kết nối cho gói tin.
3. Port của máy ảo trên self-service bridge(10) chuyển tiếp gói tin đến máy ảo đích(11) thông qua **veth pair**.


### 2.3 East-west - trên hai VXLAN khác nhau.

Thực hiện kịch bản với hai máy ảo ở hai mạng self-service khác nhau nhưng cùng một compute node.

Các máy ảo ở hai mạng self-service khác nhau muốn giao tiếp với nhau thì cần có trên một router giống nhau.

![](https://i.imgur.com/9fXsZxd.png)

Các bước thực hiện trên compute node:
1. Interface của máy ảo(1) gửi gói tin đến  port của máy ảo(2) trên provider bridge thông qua **veth pair**
2. Security group rule(3) sẽ xử lý firewall và theo dõi kết nối 
3. Self-service bridge chuyển gói tin đến VXLAN interface(4) để đóng gói gói tin sử dụng VNI 101.
4. Interface vật lý(5) được dùng bởi VXLAN interface chuyển tiếp gói tin mà đã được đóng gói qua mạng overlay(6).

Các bước sau thực hiện trên Network node:
1. Interface vật lý(7) được dùng bởi VXLAN interface trên network node chuyển tiếp gói tin đến VXLAN interface(8) để gỡ lớp đóng gói VNI 101.
2. Port của router trên self-service bridge(9) chuyển tiếp gói tin đến interface mạng self-service 1(10) trên Router namespace.
3. Router gửi gói tin đến địa chỉ IP next hop, thường là địa chỉ IP gateway của mạng self-service 2 thông qua interface self-service 2 trên router(11).
4. Router chuyển tiếp gói tin đến port trên self-service 2 bridge(12).
5. Self-service 2 bridge nhận và chuyển tiếp gói tin đến VXLAN interface(13) để đóng  gói gói tin với VNI 102.
6. Interface vật lý(14) dùng cho VXLAN  gửi packet đến compute node thông qua Overlay network ( 15 )

Trên compute node:
1. Physical interface (16) dùng cho VXLAN trên compute node nhận gói tin và gửi đến VXLAN interface ở self-server bridge (17) và gỡ đóng gói VNO 102.
2. Security group đảm nhiệm filter packet (18)
3. Port của máy ảo trên self-service bridge(19) chuyển packet đến port của máy ảo(20) thông qua **veth pair**.





## Tài liệu tham khảo.
https://docs.openstack.org/neutron/train/admin/deploy-lb-provider.html
https://docs.openstack.org/neutron/train/admin/deploy-lb-selfservice.html





## Tài liệu tham khảo.
https://docs.openstack.org/neutron/train/admin/deploy-lb-provider.html
https://docs.openstack.org/neutron/train/admin/deploy-lb-selfservice.html
