# Cách cài đặt OPS yoga trên Ubuntu 20.04

- **Mô hình cơ bản**

![image](https://user-images.githubusercontent.com/83824403/178680453-e58aa11f-bb1d-4983-adde-2f5f9f4885b7.png)

## 1.  Node controller và compute cùng làm các thao tác sau
-  sửa file `/etc/hosts` như sau:

172.16.2.129 controller
172.16.2.130 compute1

- Ping từ controller sang compute và ngược lại

![image](https://user-images.githubusercontent.com/83824403/178681634-905d330f-8f22-463e-8f52-8bd552558d0a.png)


- Check date xem controller và compute đã trùng nhau chưa. Nếu đã trùng thì không cần cài NTP sync cũng được

- Install OPS package

```
 add-apt-repository cloud-archive:yoga
 apt install nova-compute -y
 apt install python3-openstackclient -y
 ```
 
 ## 2. Node controller
 
 ### 2.1 Cài DB, Message queue, Memcached, ETCD
 
 #### 2.1.1 Cài DB
 ```
 apt install mariadb-server python3-pymysql -y
 ```
 
 - Tạo & Sửa file `/etc/mysql/mariadb.conf.d/99-openstack.cnf` như sau


```
[mysqld]
bind-address = 172.16.2.129

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```


- Restart service mysql

```
service mysql restart
```
 #### 2.1.2 Cài Message queue
 
- Tải gói 

```
apt install rabbitmq-server
```
- Thêm OPS user


```
rabbitmqctl add_user openstack phucdv

Creating user "openstack" ...
Replace RABBIT_PASS with a suitable password.
```

- Cấp phát quyền cho user OPS

```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"


Setting permissions for user "openstack" in vhost "/" ...

```
#### 2.1.3 Memcached

- Cài Memcached
```
 apt install memcached python3-memcache
```

- Sửa file `/etc/memcached.conf` 
```
-l 172.16.2.129

 - Note

Change the existing line that had -l 127.0.0.1.
```


- Restart lại service
```
service memcached restart
```
#### 2.1.4 ETCD

- Cài etcd

```
apt install etcd -y
```




- Sửa file `/etc/default/etcd` như sau
```

ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://172.16.2.129:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.2.129:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.2.129:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.16.2.129:2379"
```

- enable và restart lại service

```
systemctl enable etcd
systemctl restart etcd
```




 ### 2.2 Cài Keystone

- Tạo Keystone DB 

```
 mysql


MariaDB [(none)]> CREATE DATABASE keystone;
Grant proper access to the keystone database:

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'phucdv';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'phucdv';
```


- Tải Keystone xuống
```
apt install keystone -y
```
- Sửa file `/etc/keystone/keystone.conf` 

```

[database]
connection = mysql+pymysql://keystone:phucdv@controller/keystone

[token]
provider = fernet
Populate the Identity service database:



```

- Populate dịch vụ identify với DB:

```
 su -s /bin/sh -c "keystone-manage db_sync" keystone
```

- Khởi tạo khóa fernet

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

```

- Bootstrap dịch vụ Identity:
```
keystone-manage bootstrap --bootstrap-password phucdv \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
- Sửa file `httpd.conf` và thêm dòng
```
ServerName controller
```

- Restart httpd
```

 service apache2 restart
```

- Tạo script chứa thông tin đăng nhập vào dịch vụ identity

```
root@controller:~# vim admin-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=phucdv
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
 - Tạo domain, project, user, role

- Tạo domain: 

Mặc dù miền "mặc định" đã tồn tại từ bước khởi động keystone-quản lý trong hướng dẫn này, một cách chính thức để tạo một miền mới sẽ là:

```
openstack domain create --description "An Example Domain" example
```
 
 ![image](https://user-images.githubusercontent.com/83824403/178687875-2be15179-6c14-41ad-9710-8f28c93e4419.png)
 
 
- Tạo service project

```
openstack project create --domain default \
  --description "Service Project" service
```

- Các tác vụ thông thường (không phải quản trị viên) nên sử dụng một dự án và người dùng không có đặc quyền. Ví dụ, hướng dẫn này tạo dự án myproject và người dùng myuser.

Tạo dự án myproject:

```
openstack project create --domain default \
  --description "Demo Project" myproject
```

- Tạo user là myuser:

```
 openstack user create --domain default \
  --password-prompt myuser
```


- Tạo role là myrole

```

openstack role create myrole
```

- Thêm vai trò myrole vào dự án myproject và người dùng myuser:


```
openstack role add --project myproject --user myuser myrole

```

 
 - Nhìn chung các step trên cuối cùng như sau là được:

![image](https://user-images.githubusercontent.com/83824403/178689087-1116bda5-ae61-411b-9dbf-0316107fa096.png)



- Xác minh operation



Bỏ đặt biến môi trường OS_AUTH_URL và OS_PASSWORD tạm thời:
```

unset OS_AUTH_URL OS_PASSWORD
```

- Request authen token

```
 openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```


- Như người dùng myuser đã tạo trước đó, hãy yêu cầu mã thông báo xác thực:
  
 ```
 openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue
  
```
  
 
-  Như sau:

![image](https://user-images.githubusercontent.com/83824403/178689770-752a06bf-eb49-4be1-a40b-d0e0d025bc2c.png)


- Request an authentication token:


```
$ openstack token issue

+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2022-07-12T20:44:35.659723Z                                     |
| id         | gAAAAABWvjYj-Zjfg8WXFaQnUd1DMYTBVrKw4h3fIagi5NoEmh21U72SrRv2trl |
|            | JWFYhLi2_uPR31Igf6A8mH2Rw9kv_bxNo1jbLNPLGzW_u5FC7InFqx0yYtTwa1e |
|            | eq2b0f6-18KZyQhs7F3teAta143kJEWuNEYET-y7u29y0be1_64KYkM7E       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+

```

 
 
 
 
 
 ### 2.3 Cài Glance
 

- Khởi tạo database cho glance

```sh
mysql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'    IDENTIFIED BY 'phucdv';
exit
```

- 	Tạo thông tin xác thực dịch vụ cho glance

```sh
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```


- Ví dụ như sau

 ![image](https://user-images.githubusercontent.com/83824403/178691331-9cf0fb44-e7d0-44e2-992d-ab34f93e7e08.png)



- 	Cài đặt dịch vụ glance
```
 apt install glance -y
```

- Chỉnh sửa file cấu hình dịch vụ glance

```
vim /etc/glance/glance-api.conf


[database]  //truy cập CSDL
connection = mysql+pymysql://glance:phucdv@controller/glance

[keystone_authtoken] //thông tin truy cập keystone
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = phucdv

[paste_deploy]
flavor = keystone

[glance_store] //vị trí lưu image
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

```

- Populate vào CSDL dịch vụ glance

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

- Khởi động lại dịch vụ

```
service glance-api restart
```

-	Tải image và upload lên dịch vụ glance
```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
glance image-create --name "cirros" \
>   --file cirros-0.4.0-x86_64-disk.img \
>   --disk-format qcow2 --container-format bare \
>   --visibility=public
openstack image list
```

- Kết quả sẽ như sau

![image](https://user-images.githubusercontent.com/83824403/178711695-18006ddd-8a24-4572-9ce4-4fcc4162059d.png)



### Cài đặt dịch vụ Placement
#### Trên node controller
- Khởi tạo database cho placement
```
# mysql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost'  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%'  IDENTIFIED BY 'phucdv';
exit
```

- 	Tạo thông tin xác thực dịch vụ cho placement

```sh
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement placement
openstack endpoint create --region RegionOne \
>   placement public http://controller:8778
openstack endpoint create --region RegionOne \
>   placement internal http://controller:8778
openstack endpoint create --region RegionOne \
>   placement admin http://controller:8778
```

- 	Cài đặt dịch vụ placement
```
apt install placement-api -y 
```

- Chỉnh sửa file cấu hình dịch vụ placement

```
 vim  /etc/placement/placement.conf

[placement_database] //truy cập CSDL
connection = mysql+pymysql://placement:phucdv@controller/placement

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = phucdv
```


- Populate vào CSDL placement

```
su -s /bin/sh -c "placement-manage db sync" placement
```

- Khởi động lại dịch vụ

```sh
service apache2 restart
```

 
 
 
 
 
 
 
 
 
 ### 2.4 Cài Nova
 

#### Trên node Controller

- Khởi tạo database cho nova


```
 mysql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' 
IDENTIFIED BY 'phucdv';
exit
```

- 	Tạo thông tin xác thực dịch vụ cho nova

```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

- 	Cài đặt dịch vụ nova
```
 apt install nova-api nova-conductor nova-novncproxy nova-scheduler -y
```

- Chỉnh sửa file cấu hình dịch vụ nova

```
vim /etc/nova/nova.conf


[DEFAULT]
transport_url = rabbit://openstack:phucdv@controller:5672/
my_ip = 172.16.2.129

[api_database]
connection = mysql+pymysql://nova:phucdv@controller/nova_api

[database]
connection = mysql+pymysql://nova:phucdv@controller/nova

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = phucdv

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]  //configure the location of the Image service API
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]  // truy cập placement
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = phucdv
```


- Populate vào CSDL nova-api, nova và đăng ký database cell0 và cell1

```
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

- Khởi động lại dịch vụ

```
service nova-api restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
```


#### Trên node compute

- 	Cài đặt dịch vụ nova-compute
```
apt -y install nova-compute
```

- Chỉnh sửa file cấu hình dịch vụ nova

```
vim /etc/nova/nova.conf


[DEFAULT]
transport_url = rabbit://openstack:phucdv@controller:5672/
my_ip = 172.16.2.130

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = phucdv

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://172.16.2.129:6080/vnc_auto.html

[glance]  //configure the location of the Image service API
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]  // truy cập placement
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = phucdv
```

- Cấu hình nova-compute

```
vim /etc/nova/nova-compute.conf

[libvirt]
virt_type = qemu
```


- Khởi động lại dịch vụ

```
service nova-compute restart
```

#### Trên node Controller (Thêm node compute vào cell database)

![image](https://user-images.githubusercontent.com/83824403/178720080-1e35e4c7-fac7-4024-98f7-d9ee0424efdc.png)

```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```


 
 
 
 ### 2.5 Cài Neutron
 
 
 #### Trên node Controller

- Khởi tạo database cho neutron

```
mysql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'  IDENTIFIED BY 'phucdv';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'phucdv';
exit
```

- 	Tạo thông tin xác thực dịch vụ cho neutron

```
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron  --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```



![image](https://user-images.githubusercontent.com/83824403/178871051-396824b6-a3e4-4ca1-b314-62d183d78d77.png)



- 	Cài đặt dịch vụ neutron

```
 apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
```



- Cấp quyền truy cập cho admin

```
. admin-openrc
openstack user create --domain default --password-prompt neutron
```

- Add the admin role to the neutron user:
```
 openstack role add --project service --user neutron admin
 ```
 

- Tạo neutron service entity:
```
openstack service create --name neutron  --description "OpenStack Networking" network
```


**`Provider networks`**


- Chỉnh sửa file cấu hình dịch vụ netron

```
vim /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:phucdv@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true


[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = phucdv

### NOTE: Comment out or remove any other connection options in the [database] section.

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = phucdv

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```

 vim /etc/neutron/metadata_agent.ini

[DEFAULT]
nova_metadata_host =172.16.2.129
metadata_proxy_shared_secret = phucdv

```
```
 vim /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[securitygroup]
enable_ipset = true
```


- Sửa file /etc/neutron/plugins/ml2/linuxbridge_agent.ini


```
[linux_bridge]
physical_interface_mappings = provider:ens37

[vxlan]
enable_vxlan = false

[securitygroup]

enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

- Đảm bảo kernel Linux của hỗ trợ bộ lọc cầu nối mạng bằng cách xác minh tất cả các giá trị sysctl sau được đặt thành 1:

``` 
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables

```

- Cấu hình DHCP agent


```
vim /etc/neutron/dhcp_agent.ini

[DEFAULT]

interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```


- Cấu hình metadata agent


```
 vim /etc/neutron/metadata_agent.ini



[DEFAULT]

nova_metadata_host = 172.16.2.129
metadata_proxy_shared_secret = phucdv
```


- Định cấu hình compute để sử dụng dịch vụ Mạng 


```
vim /etc/nova/nova.conf

[neutron]

auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = phucdv
service_metadata_proxy = true
metadata_proxy_shared_secret = phucdv
```





- Populate the database:

```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

*`NOTE: Database population occurs later for Networking because the script requires complete server and plug-in configuration files.`*

- Restart the Compute API service:

```
 service nova-api restart
```
 
 
 - Restart networking service


```
 service neutron-server restart
 service neutron-linuxbridge-agent restart
 service neutron-dhcp-agent restart
 service neutron-metadata-agent restart


HOẶC CHẠY: service neutron* restart
```



#### Trên node compute

- 	Cài đặt dịch vụ neutron
```
apt install neutron-linuxbridge-agent -y
```

- Chỉnh sửa file cấu hình dịch vụ netron

```
vim /etc/neutron/neutron.conf

[DEFAULT]
auth_strategy = keystone
transport_url = rabbit://openstack:phucdv@172.16.2.129

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = phucdv

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

- Cấu hình compute service để có thể SD network service

```
vim /etc/nova/nova.conf 


[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = phucdv

```

- Cuối cùng, restart lại các service liên quan


```
service nova-compute restart

service neutron-linuxbridge-agent restart

```


#### Tạo mạng với provider network ở controller


```
. admin-openrc

openstack network create  --share --external --provider-physical-network provider --provider-network-type flat provider
  
openstack subnet create --network provider --allocation-pool start=192.168.1.100,end=192.168.1.200 --dns-nameserver 8.8.8.8 --gateway 172.168.1.0 --subnet-range 192.168.0.0 provider
```  
  
### 2.6 Cài Dashboard Horizon


- Tải OPS- dashboard

```
apt install openstack-dashboard -y
```
- Sửa file cấu hình theo các đề mục sau
```
vim /etc/openstack-dashboard/local_settings.py 

OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*', '172.16.2.129']


SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': '172.16.2.129:11211',
    }
}

```
- Những dòng khác chúng ta sẽ commnent hết lại
-  Sau đó làm theo các bước sau


- Enable the Identity API version 3:

```
OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
Configure API versions:

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}


OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"


OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"


OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}


TIME_ZONE = "TIME_ZONE"
Replace TIME_ZONE with an appropriate time zone identifier. For more information, see the list of time zones.

```
- Sửa file như sau nếu nó chưa có sẵn

```
vim /etc/apache2/conf-available/openstack-dashboard.conf if not included.



WSGIApplicationGroup %{GLOBAL}
```

```
 systemctl reload apache2.service
``` 
### Cuối cùng restart lại service và vào web duyệt theo đường dẫn `172.16.2.129/Horizon/`


- Tạo instance ...
 
**`=>> Kết quả cuối cùng như sau:`**

![image](https://user-images.githubusercontent.com/83824403/178881294-956caad6-d742-42b5-bf64-9c7a5d0765bf.png)




- **Ngoài ra có thể tạo mạng self-service, nhìn chung nó như sau:**

![image](https://user-images.githubusercontent.com/83824403/178882097-faee749b-6c24-4d13-a7ec-817f9a2d50cf.png)


