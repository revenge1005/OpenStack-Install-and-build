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
apt -y install neutron-server neutron-plugin-ml2 python3-neutronclient 
```

#### (2) /etc/neutron/neutron.conf 파일 수정
```
mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
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
lock_path = \$state_path/tmp
EOF
```
```
chmod 640 /etc/neutron/neutron.conf
chgrp neutron /etc/neutron/neutron.conf
```

#### (3) /etc/neutron/plugins/ml2/ml2_conf.ini 파일 수정

```
vi /etc/neutron/plugins/ml2/ml2_conf.ini
```
```bash
[ml2]
# Neutron, ml2 네트워크 드라이버가 지원하는 목록
type_drivers = flat,vlan,gre,vxlan

# tenant_network로 할당되는 네트워크 유형의 목록을 의미
tenant_network_types = 

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
```
vim /etc/nova/nova.conf
```
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

<br>

----

## 2-1. Network 노드 - 커널 설정
```
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sed -i 's/#net.ipv4.conf.default.rp_filter=1/net.ipv4.conf.default.rp_filter=0/' /etc/sysctl.conf
sed -i 's/#net.ipv4.conf.all.rp_filter=1/net.ipv4.conf.all.rp_filter=0/' /etc/sysctl.conf

sysctl -p
```

## 2-2. Network 노드 - Neutron 패키지 설치 및 설정

#### (1) 설치
#### - 메뉴얼 대로 하면, /etc/neutron/fwaas_driver.ini 파일이 없다면서 neutron-l3-agent가 실행되지 않는다.
#### - 찾아보니 해당 파일은 Ubuntu 환경에서만 누락되어있는 것 같다. (https://github.com/cloudbase/salt-openstack/issues/23)
#### - 해결 방법은 따로 설치하면 (apt -y install python3-neutron-fwaas) 된다. (아래 내용에 추가되어 있다.)
```
apt -y install neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent python3-neutronclient python3-neutron-fwaas
```

#### (2) /etc/neutron/neutron.conf 파일 수정
```
mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org
```
```bash
cat <<EOF >/etc/neutron/neutron.conf 
[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = True
# RabbitMQ connection info
transport_url = rabbit://openstack:RABBIT_PASS@controller

[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_DBPASS

[oslo_concurrency]
lock_path = \$state_path/tmp
EOF
```
```
chmod 640 /etc/neutron/neutron.conf
chgrp neutron /etc/neutron/neutron.conf
```

#### (3) /etc/neutron/l3_agent.ini 파일 수정
#### - L3 에이전트가 사용하는 파일, 셀프서비스 가상 네트워크에 라우팅과 NAT 서비스를 제공하는 역할을 함
```
vim /etc/neutron/l3_agent.ini
```
```
#line16
interface_driver = linuxbridge
```

#### (4) /etc/neutron/dhcp_agent.ini 파일 수정
#### - DHCP 에이전트가 사용하는 파일, 가상 네트워크에 DHCP 서비스를 제공
```
vim /etc/neutron/dhcp_agent.ini 
```
```
[DEFAULT]
# 가상 네트워크 인터페이스를 관리하기 위해 사용되는 드라이버 (openvswitch, linuxbridge)
interface_driver = linuxbridge

# DHCP 서버를 관리하기 위해 사용되는 드라이버 정의
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq

# 독립된 metadata 기능 사용함으로 프로바이더 네트워크에서 인스턴스가 네트워크를 통해 metadata를 접속할 수 있게 허용
enable_isolated_metadata = true
```

#### (5) /etc/neutron/metadata_agent.ini 파일 수정
#### - Metadata 에이전트가 사용하는 파일, 인증정보와 같은 설정정보를 인스턴스에게 제공
```
vim /etc/neutron/metadata_agent.ini
```
```
[DEFAULT] 
# nova-api 서버의 IP 주소
nova_metadata_host = controller

# NOVA-METADATA 서버 비밀키 설정
metadata_proxy_shared_secret = metadata_secret

# uncomment and specify Memcache server
memcache_servers = controller:11211
```

#### (6) /etc/neutron/plugins/ml2/ml2_conf.ini 파일 수정
```
vim /etc/neutron/plugins/ml2/ml2_conf.ini
```
```
[ml2]
# OK with no value for [tenant_network_types] now (set later if need)
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = 
mechanism_drivers = linuxbridge,openvswitch,l2population
extension_drivers = port_security

#enable_ipset = True
```
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

#### (7) /etc/neutron/plugins/ml2/linuxbridge_agent.ini 파일 수정
```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```
```
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = True
enable_ipset = True

# line 284:
local_ip = 192.168.219.130
```

#### (8) Neutron restart/enable
```
systemctl restart neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent neutron-linuxbridge-agent
systemctl enable neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent neutron-linuxbridge-agent
```

#### (9) Controller 노드 - 결과 확인
```
openstack network agent list
```

<br>

----