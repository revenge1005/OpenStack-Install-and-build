## 1-1. Horizon 설치

#### - https://docs.openstack.org/horizon/ussuri/install/install-ubuntu.html

#### - Horizon의 설치 및 설정은 "Controller Node"에서만 작업한다.

#### (1) 설치
```
apt install openstack-dashboard -y
```

#### (2) 설정
```
vim /etc/openstack-dashboard/local_settings.py


### 대시보드에 엑세스 할 수 있는 호스트 명시 
### *로 할 경우 모든 호스트 허용되며, 여러 호스트를 명시할때는 , 로 구분
ALLOWED_HOSTS = ['*', 'localhost' ]


### openstack api version 명시(identity : 3, image : 2, volume : 2)
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}


### keystone Multidomain 지원 설정
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True


### 기본 설정으로 사용할 도메인 설정(대시보드를 통해 사용자 생성 시 기본값으로 사용됨)
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'


### memcached 세션 저장소 구성 
### 세션 엔진으로 장고를 사용
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

### openstack 호스트가 controller 임을 명시
OPENSTACK_HOST = "controller"


### identity Service API의 URL 명시(버전 3 사용)
OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST

### 기본 설정으로 사용할 역할 설정(대시보드를 통해 사용자 생성 시 기본값으로 사용됨)
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

### 네트워킹을 옵션 1로 설정한 경우 3계층 서비스들에 대해 비활성화
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Seoul"
```
#### (3) /etc/apache2/conf-available/openstack-dashboard.conf 파일에서 기본적으로 필요한 WSGIApplicationGroup %{GLOBAL} 구문이 빠져있다면 추가
```
WSGIApplicationGroup %{GLOBAL}
```

#### (4) 서비스 재시작
```
systemctl restart apache2 memcached
```