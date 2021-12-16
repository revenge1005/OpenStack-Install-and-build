## 1-1. 데이터베이스 설정

#### - https://docs.openstack.org/glance/ussuri/install/install-ubuntu.html#install-and-configure-components

#### - Glance의 설치 및 설정은 "Controller Node"에서만 작업한다.
```
mysql -u root -p
```
```
create database glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
flush privileges;
exit
```

## 1-2. Glance 사용자, 서비스, EndPoint 생성

#### (1) 생성
```
openstack user create --domain default --project service --password password glance

openstack role add --project service --user glance admin

openstack service create --name glance --description "Openstack Image" image

openstack endpoint create --region RegionOne image public http://controller:9292

openstack endpoint create --region RegionOne image internal http://controller:9292

openstack endpoint create --region RegionOne image admin http://controller:9292
```

#### (2) 확인
```
openstack user list

openstack service list

openstack endpoint list
```

## 1-3. Glance 패키지 설치

#### (1) 설치
```
apt install glance -y
```

#### (2) 설정 - 1
```
vim /etc/glance/glance-api.conf 

### 데이터베이스 액세스 구성
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

### 인즌 서비스 액세스 구성
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

### 인증 서비스 액세스 구성
[paste_deploy]
# ...
flavor = keystone

### 파일 시스템 저장소, 이미지 파일 위치 구성
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
```
chmod 644 /etc/glance/glance-api.conf

chown glance.glance /etc/glance/glance-api.conf
```

#### (2) 설정 - 2
```
mv /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak

vim /etc/glance/glance-registry.conf 

### 데이버베이스 액세스 구성 
[database] 
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

### 인증 서비스 액세스 구성
[keystone_authtoken] 
www_authenticate_uri = http://controller:5000 
auth_url = http://controller:5000 
memcached_servers = controller:11211 
auth_type = password 
project_domain_name = Default 
user_domain_name = Default 
project_name = service 
username = glance 
password = GLANCE_PASS

### 인증 서비스 액세스 구성
[paste_deploy]  
flavor = keystone
```
```
chmod 644 /etc/glance/glance-registry.conf 

chown glance.glance /etc/glance/glance-registry.conf 
```

#### (3) glance-manage db_sync [DB이름] 명령을 통해 image service 데이터베이스 초기 구성
```
su -s /bin/sh -c "glance-manage db_sync" glance
```

#### (3-1) 결과 확인
```
mysql -u root -p glance
```

#### (4) 서비스 restart/enable
```
systemctl enable openstack-glance-api.service \ openstack-glance-registry.service

systemctl restart openstack-glance-api.service \ openstack-glance-registry.service
```

#### (5) Glance 서비스 로그 위치 확인
```
ls -l /var/log/glance
```