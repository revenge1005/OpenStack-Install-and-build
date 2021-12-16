## 1-1. Controller 노드 - 데이터베이스 설정

#### - https://docs.openstack.org/nova/ussuri/install/controller-install-ubuntu.html#prerequisites

```
mysql -u root -p
```
```
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
```
```
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

grant all privileges on nova_placement.* to nova@'localhost' identified by 'NOVA_DBPASS';

grant all privileges on nova_placement.* to nova@'%' identified by 'NOVA_DBPASS';
```

## 1-2. Controller 노드 - Nova 사용자, 서비스, EndPoint 생성

#### (1) 생성
```
openstack user create --domain default --project service --password password nova

openstack role add --project service --user nova admin

openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1

openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1

openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```
```
openstack user create --domain default --project service --password password placement

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
```

## 1-3. Controller 노드 - Nova 패키지 설치 및 설정

#### (1) 설치
```
apt -y install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-placement-api nova-scheduler nova-cert python3-novaclient
```

#### (2) 설정
```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
vim /etc/nova/nova.conf

[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 192.168.219.110
use_neutron = true
log_dir = /var/log/nova
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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
username = nova
password = NOVA_PASS

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

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
```
```
chmod 640 /etc/nova/nova.conf
chgrp nova /etc/nova/nova.conf
```

## 1-4. 데이터베이스 테이블 생성
```
su -s /bin/bash nova -c "nova-manage api_db sync"

su -s /bin/bash nova -c "nova-manage cell_v2 map_cell0 --database_connection mysql+pymysql://nova:password@controller/nova_cell0"

su -s /bin/bash nova -c "nova-manage db sync"

su -s /bin/bash nova -c "nova-manage cell_v2 create_cell --name cell1 --database_connection mysql+pymysql://nova:password@controller/nova --transport-url rabbit://openstack:password@controller:5672"
```

## 1-5. 서비스 restart/enable
```
systemctl restart nova-api nova-consoleauth nova-scheduler nova-conductor nova-novncproxy

systemctl enable nova-api nova-consoleauth nova-scheduler nova-conductor nova-novncproxy
```

<br>

----

## 2-1. Compute 노드 - Nova 서비스 설치 및 설정

#### - https://docs.openstack.org/nova/ussuri/install/compute-install-ubuntu.html#install-and-configure-components

```
```