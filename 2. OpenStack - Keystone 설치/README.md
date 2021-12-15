## 1-1. 데이터베이스 설정

#### - https://docs.openstack.org/keystone/victoria/install/keystone-install-ubuntu.html

#### - Keystone의 설치 및 설정은 "Controller Node"에서만 작업한다.

```bash
mysql -u root -p
```

```
create database keystone default character set utf8 default collate utf8_general_ci;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone 계정 비밀번호';

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone 계정 비밀번호';

flush privileges;
exit
```

## 1-2. Keystone 설치

#### (1) keystone 설치 
```
apt install keystone -y
```

#### (2) /etc/keystone/keystone.conf 을 통해 keystone 설정 수정
```bash
vim /etc/keystone/keystone.conf

[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider = fernet
```

#### (3) indentity service database 내용 채우기
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

#### (4) keystone-manage fernet_setup 명령을 실행시켜 fernet key 저장소 초기화
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```
#### (4-1) 확인
```
ls -l /etc/keystone/fernet-keys/
```

#### (5) keystone-manage credential_setup 명령을 실행시켜 fernet key 암호화
```
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
#### (5-1) 확인
```
ls -l /etc/keystone/credential-keys
```

#### (6) indentity 서비스에 대해 bootstrap 적용
```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```