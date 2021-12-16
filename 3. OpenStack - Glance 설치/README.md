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