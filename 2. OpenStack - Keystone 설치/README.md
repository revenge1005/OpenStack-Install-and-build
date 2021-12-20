## 1-1. 데이터베이스 설정

#### - https://docs.openstack.org/keystone/victoria/install/keystone-install-ubuntu.html

#### - Keystone의 설치 및 설정은 "Controller Node"에서만 작업한다.

```bash
mysql -u root -p
```

```
create database keystone default character set utf8 default collate utf8_general_ci;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';

flush privileges;
exit
```

## 1-2. Keystone 패키지 설치 및 설정

#### (1) keystone 설치 
```
apt -y install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 python3-oauth2client
```

#### (2) /etc/keystone/keystone.conf 을 통해 keystone 설정 수정
```bash
vim /etc/keystone/keystone.conf

# line 436: uncomment and specify Memcache Server
memcache_servers = controller:11211

# line 594: change to MariaDB connection info
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

# line 2508: uncomment
[token]
provider = fernet
```

#### (3) indentity service database 내용 채우기
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

#### (4) keystone-manage을 통해서 fernet key 저장소 초기화
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
#### (5) 결과 확인
```
ls -l /etc/keystone/fernet-keys/

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

## 1-3. Apache HTTP Server 구성

#### (1) http config 구성
```
vim /etc/apache2/apache2.conf

# line 70: specify server name
ServerName controller
```

### (2) HTTP 서비스가 부팅 시 시작되도록 등록하고 서비스를 시작한다.
```
systemctl enable apache2
systemctl restart apache2
```

## 1-4. 환경 변수 설정

#### (1) Start the Apache HTTP service and configure it to start when the system boots:
#### (2) Configure the administrative account by setting the proper environmental variables:
```
vim ~/keystonerc

export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

#### (3)
```
chmod 600 ~/keystonerc
source ~/keystonerc
echo "source ~/keystonerc " >> ~/.bash_profile

openstack token issue
```

#### (4) 각종 서비스들이 설정될 service project 생성
```
openstack project create --domain default --description "Service Project" service
```

#### (5) 결과 확인
```
openstack project list

openstack domain list

openstack role list

openstack endpoint list
```