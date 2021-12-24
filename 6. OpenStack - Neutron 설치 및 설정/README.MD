## 1-1. Controller 노드 - 데이터베이스 설정

#### - https://docs.openstack.org/neutron/ussuri/install/controller-install-ubuntu.html#configure-networking-options

```
mysql -u root -p
```
```
CREATE DATABASE neutron;

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';

flush privileges;

exit
```

## 1-2. Controller 노드 - Nova 사용자, 서비스, EndPoint 생성

#### (1) 생성
```
openstack user create --domain default --project service --password NEUTRON_DBPASS neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://controller:9696

openstack endpoint create --region RegionOne network internal http://controller:9696

openstack endpoint create --region RegionOne network admin http://controller:9696
```

#### (2) 확인
```
openstack user list

openstack service list

openstack endpoint list | grep neutron
```

## 1-3. Controller 노드 - Neutron 패키지 설치 및 설정

#### (1) 설치
```
apt -y install nova-api nova-conductor nova-scheduler nova-novncproxy placement-api python3-novaclient
```

#### (2) /etc/neutron/neutron.conf 파일 수정
```
mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak

vim /etc/neutron/neutron.conf
```
```bash
cat <<EOF >/etc/neutron/neutron.conf
[DEFAULT]
# Neutron에서 사용할 핵심 플로그인, ml2 활성화
core_plugin = ml2
# Neutron에서 사용할 서비스 플로그인, router 활성화 그외에도 firewall, vpnaas 등 추가 가능
service_plugins = router
# 인증 방법 - Keystone 으로 설정
auth_strategy = keystone
state_path = /var/lib/neutron
# overlapping ip 활성화
allow_overlapping_ips = True
dhcp_agent_notification = True
# nova에 네트워크 토폴로지 변경을 알리도록 설정
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
# RabbitMQ connection info
transport_url = rabbit://openstack:RABBIT_PASS@controller

[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

# Keystone auth info
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_DBPASS

# MariaDB connection info
[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

# Nova auth info
[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_DBPASS

[oslo_concurrency]
lock_path = $state_path/tmp
```
```
chmod 640 /etc/neutron/neutron.conf
chgrp neutron /etc/neutron/neutron.conf
```

#### (3) /etc/neutron/plugins/ml2/ml2_conf.ini 파일 수정

```
vi /etc/neutron/metadata_agent.ini
```
```bash
[ml2]
# Neutron, ml2 네트워크 드라이버가 지원하는 목록
type_drivers = flat,vlan,gre,vxlan
# tenant_network로 할당되는 네트워크 유형의 목록을 의미
tenant_network_type = 
# 대규모의 네트워크에서 트래픽을 보내기 위해 사용되는 드라이버
# openvswitch, linuxbridge가 주로 사용되며, 여기서는 linuxbridge를 사용한다.
mechanism_drivers = linuxbridge,openvswitch,l2population
# ml2가 지원하는 확장 드라이버 가상머신에서 패킷 필터링 기능을 허용하기 위해 사용
extension_drivers = port_security
```
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

#### (4) /etc/nova/nova.conf 파일 수정

```bash
# add follows into [DEFAULT] section
[DEFAULT]
use_neutron = True
linuxnet_interface_driver = nova.network.linux_net.LinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

# add follows to the end : Neutron auth info
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_DBPASS
service_metadata_proxy = True
metadata_proxy_shared_secret = metadata_secret
```

#### (5) 데이터베이스 테이블 추가 및 Neutron 재시작

```
su -s /bin/bash neutron -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head"

systemctl restart neutron-server nova-api
systemctl enable neutron-server 
```