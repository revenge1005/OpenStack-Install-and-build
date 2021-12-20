## 1-1. Controller 노드 - 데이터베이스 설정

#### - https://docs.openstack.org/nova/ussuri/install/controller-install-ubuntu.html#prerequisites

```
mysql -u root -p
```
```
CREATE DATABASE nova_api;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

CREATE DATABASE nova;

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

CREATE DATABASE placement;

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';

flush privileges;

exit
```

## 1-2. Controller 노드 - Nova 사용자, 서비스, EndPoint 생성

#### (1) 생성
```
openstack user create --domain default --project service --password NOVA_DBPASS nova

openstack role add --project service --user nova admin

openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```
```
openstack user create --domain default --project service --password PLACEMENT_DBPASS placement

openstack role add --project service --user placement admin

openstack service create --name placement --description "Compute Placement API" placement

openstack endpoint create --region RegionOne placement public http://controller:8778

openstack endpoint create --region RegionOne placement internal http://controller:8778

openstack endpoint create --region RegionOne placement admin http://controller:8778
```

#### (2) 확인
```
openstack user list

openstack service list

openstack endpoint list | grep nova

openstack endpoint list | grep placement
```

## 1-3. Controller 노드 - Nova 패키지 설치 및 설정

#### (1) 설치
```
apt -y install nova-api nova-conductor nova-scheduler nova-novncproxy placement-api python3-novaclient
```

#### (2) 설정 - 1
```
mv /etc/nova/nova.conf /etc/nova/nova.conf.org

vim /etc/nova/nova.conf

# create new
[DEFAULT]
# define own IP address
my_ip = 192.168.56.110
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
log_dir = /var/log/nova
# RabbitMQ connection info
transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/

[api]
auth_strategy = keystone

# Glance connection info
[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = $state_path/tmp

# MariaDB connection info
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS

[wsgi]
api_paste_config = /etc/nova/api-paste.ini
```
```
chgrp nova /etc/nova/nova.conf
chmod 640 /etc/nova/nova.conf
```

#### (3) 설정 - 2
```
mv /etc/placement/placement.conf /etc/placement/placement.conf.org
vi /etc/placement/placement.conf

[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
```
```
chmod 640 /etc/placement/placement.conf
chgrp placement /etc/placement/placement.conf
```

## 1-4. 데이터베이스 테이블 생성
```
su -s /bin/bash placement -c "placement-manage db sync"

su -s /bin/sh -c "nova-manage api_db sync" nova

su -s /bin/sh -c "nova-manage cell_v2 map_cell0 --database_connection mysql+pymysql://nova:NOVA_PASS@controller/nova_cell0" nova

su -s /bin/sh -c "nova-manage cell_v2 create_cell --name cell1 --database_connection mysql+pymysql://nova:NOVA_PASS@192.168.56.110/nova \
--transport-url rabbit://openstack:NOVA_PASS@192.168.56.110:5672" nova

su -s /bin/sh -c "nova-manage db sync" nova

su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

## 1-5. 서비스 restart/enable
```
systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy apache2

systemctl enable nova-api nova-scheduler nova-conductor nova-novncproxy

openstack compute service list
```

<br>

----

## 2-1. Compute 노드 - Nova 서비스 설치 및 설정

#### - https://docs.openstack.org/nova/ussuri/install/compute-install-ubuntu.html#install-and-configure-components

#### (1) 설치
```
apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils nova-compute-kvm python3-openstackclient nova-api-metadata nova-compute python3-novaclient -y
```

#### (2) 설정 
```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak

vim /etc/nova/nova.conf

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 192.168.56.120

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS

[wsgi]
api_paste_config = /etc/nova/api-paste.ini
```
```
chgrp nova /etc/nova/nova.conf
chmod 640 /etc/nova/nova.conf
```

## 2-2 Compute 노드 - Nova 서비스 실행
```
### 하드웨어 가속 사용 가능 여부 확인
$ egrep -c '(vmx|svm)' /proc/cpuinfo

### 1 혹은 그 이상일 경우 사용가능이므로 더이상의 설정 필요없음
### 0일 경우 사용할 수 없으므로 /etc/nova/nova.conf 에서 [libvirt] 섹션에서 가상머신 타입 qemu로 수정해야함
vi /etc/nova/nova.conf

[libvirt] 
# ... 
virt_type = qemu
```
```
modprobe vhost_net

lsmod | grep vhost
```
```
echo vhost_net >> /etc/modules

systemctl restart nova-compute

systemctl enable nova-compute
```


<br>

----

## 3-1. Controller 노드 - 결과 확인
```
su -s /bin/bash nova -c "nova-manage cell_v2 discover_hosts --verbose"

openstack compute service list

openstack hypervisor list
```