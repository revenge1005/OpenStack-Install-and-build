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