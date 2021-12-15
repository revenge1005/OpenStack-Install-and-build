
# 1-1. (모든 노드 해당) - 각 노드의 호스트 정보를 /etc/hosts에 작성

```
cat <<EOF >>/etc/hosts

# OpenStack Nodes IP adresses.
192.168.56.110  controller
192.168.56.120  compute
192.168.56.130  network
EOF
```

# 1-2. (모든 노드 해당) 방화벽 비활성화

```
ufw disable
ufw status
```

# 2-1. (Controller Node) - NTP 설치

## - OpenStack은 여러 노드로 구성되고 각각의 노드는 기본적으로 단말기를 분리하여 구성하는 것을 추천한다. 

## - 이때 NTP를 통해 각 단말기의 시간 정보를 동기화하여 보다 정확한 서비스가 가능하도록 하는 것이 좋다.

```
apt-get install chrony -y
```

```bash
vim /etc/chrony/chrony.conf 

### 기존 값 주석 처리
#pool ntp.ubuntu.com        iburst maxsources 4
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2

### 새로 추가
server 2.kr.pool.ntp.org iburst
server 1.asia.pool.ntp.org iburst
server 3.asia.pool.ntp.org iburst

### 허용 범위
allow 192.168.56/24
```

```
systemctl restart chrony
systemctl enable chrony

chronyc sources
```

# 2-2. (Compute, Network Node) - NTP 설치

```
apt-get install chrony -y
```
```bash
vim /etc/chrony/chrony.conf 

### 기존 값 주석 처리
#pool ntp.ubuntu.com        iburst maxsources 4
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2

### 추가
server controller iburst
```

```
systemctl restart chrony
systemctl enable chrony

chronyc sources
```

# 3-1. (모든 노드 해당) - OpenStack 패키지 설치

## - OpenStack Rockey 버전을 사용하기 위해 패키지 설치가 필요하다.

## https://docs.openstack.org/install-guide/environment-packages-ubuntu.html

## (1) OpenStack Victoria for Ubuntu 20.04 LTS:
```
add-apt-repository cloud-archive:victoria
```

## (2)
```
apt update
apt upgrade -y 
```

## (3)
```
apt install python3-openstackclient -y
```

# 4-1. (Controller Node) - SQL Databases 설정

## - 서비스들의 정보 저장을 위해 데이터베이스를 컨트롤러에 구성해야 한다.

## (1) As of Ubuntu 20.04, install the packages:
```
apt install mariadb-server python3-pymysql -y
```

## (2) Create and edit the /etc/mysql/mariadb.conf.d/99-openstack.cnf file and complete the following actions:
```
cat <<EOF >>/etc/mysql/mariadb.conf.d/99-openstack.cnf

[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```

## (3) Restart the database service:
```
service mysql restart
systemctl enable mysql
```

## (4) Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account
```
mysql_secure_installation
```

# 5-1. (Controller Node) - RebbitMQ (메시지 큐) 설정

## - 서비스 간 상호작용에 사용될 메시지 큐를 컨트롤러 노드에 구성해야 한다.

## https://docs.openstack.org/install-guide/environment-messaging-ubuntu.html

## (1) Install the package:
```
apt install rabbitmq-server -y
```

## (2) Add the openstack user:
```
rabbitmqctl add_user openstack RABBIT_PASS
```

## (3) Permit configuration, write, and read access for the openstack user:
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## (4) 서비스 재시작 및 enable
```
systemctl restart rabbitmq-server
systemctl enable rabbitmq-server
```

# 6-1 (Controller Node) - Memcached 설정

## - Identiry 서비스 인증에서 토큰을 캐싱하기 위해 컨트롤러 노드에 구성한다.

## https://docs.openstack.org/install-guide/environment-memcached-ubuntu.html

## (1) For Ubuntu versions prior to 20.04 use:
```
apt install memcached python3-memcache -y
```

## (2) Edit the /etc/memcached.conf file and configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:
```bash
vim /etc/memcached.conf

-l 192.168.56.110
```
## (3) Restart the Memcached service:
```
systemctl restart memcached
systemctl enable memcached
```