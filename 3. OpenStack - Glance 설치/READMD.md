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

