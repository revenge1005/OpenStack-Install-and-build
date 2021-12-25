## 1-1. Controller 노드 - 데이터베이스 설정

#### - https://docs.openstack.org/cinder/ussuri/install/cinder-controller-install-ubuntu.html#install-and-configure-components

#### (1) Cinder DB 생성
```
mysql -u root -p

CREATE DATABASE cinder;

GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';

GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```

#### (2) Cinder 사용자, 서비스, Endpoint 생성
```
openstack user create --domain default --project service --password CINDER_PASS cinder

openstack role add --project service --user cinder admin

openstack service create --name cinderv1 --description "OpenStack Block Storage" volumev1

openstack endpoint create --region RegionOne volumev1 public http://controller:8776/v1/%\(project_id\)s

openstack endpoint create --region RegionOne volumev1 internal http://controller:8776/v1/%\(project_id\)s

openstack endpoint create --region RegionOne volumev1 admin http://controller:8776/v1/%\(project_id\)s

openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2

openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
```

#### (3) 결과 확인
```
openstack user list

openstack service list

openstack endpoint list | grep cinder
```

## 1-2. Controller 노드 - Cinder 패키지 설치 및 설정

#### (1) 설치
```
apt install -y cinder-api cinder-scheduler python3-cinderclient
```

#### (2) /etc/cinder/cinder.conf 파일 수정
```
mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
```
```bash
cat <<EOF >/etc/cinder/cinder.conf
[DEFAULT]
state_path = /var/lib/cinder
auth_strategy = keystone
# IP address
my_ip = 192.168.56.110
# RabbitMQ connection info
transport_url = rabbit://openstack:RABBIT_PASS@controller
endpoint_v1_api = True
endpoint_v2_api = True

# MariaDB connection info
[database]
connection = mysql+pymysql://cinder:CINDER_PASS@controller/cinder

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = \$state_path/tmp
EOF
```
```
chmod 640 /etc/cinder/cinder.conf
chgrp cinder /etc/cinder/cinder.conf
```

#### (3) Cinder DB - Table 생성
```
su -s /bin/bash cinder -c "cinder-manage db sync"
```

#### (4) Cinder 서비스 restart/enable
```
systemctl restart apache2 cinder-scheduler

systemctl enable cinder-scheduler
```

#### (5) set environment variable
```
echo "export OS_VOLUME_API_VERSION=3" >> ~/keystonerc

source ~/keystonerc
```

#### (6) 결과 확인
```
openstack volume service list
```

<br>

----

## 2-1. Network 노드 - Cinder 패키지 설치 및 설정

#### (1) 설치
```
apt -y install cinder-volume python3-mysqldb
```

#### (2) /etc/cinder/cinder.conf 파일 수정
```
mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
```
```bash
cat <<EOF >/etc/cinder/cinder.conf
[DEFAULT]
# IP address
my_ip = 192.168.56.130
state_path = /var/lib/cinder
auth_strategy = keystone
endpoint_v1_api = True
endpoint_v2_api = True
# RabbitMQ connection info
transport_url = rabbit://openstack:password@controller
# Glance connection info
glance_api_servers = http://controller:9292
# OK with empty value now
enabled_backends = lvm

# MariaDB connection info
[database]
connection = mysql+pymysql://cinder:CINDER_PASS@controller/cinder

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = $state_path/tmp
EOF
```
```
chmod 644 /etc/cinder/cinder.conf
chgrp cinder /etc/cinder/cinder.conf
```

#### (3) Cinder 서비스 restart/enable
```
systemctl restart cinder-volume
systemctl enable cinder-volume
```

#### (4) Controller 노드 - 결과 확인
```
openstack volume service list
```