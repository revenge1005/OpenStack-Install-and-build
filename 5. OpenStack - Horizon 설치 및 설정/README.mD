## 1-1. Horizon 패키지 설치 및 설정

#### - https://docs.openstack.org/horizon/ussuri/install/install-ubuntu.html

#### - Horizon의 설치 및 설정은 "Controller Node"에서만 작업한다.

#### (1) 설치
```
apt install openstack-dashboard -y
```

#### (2) 설정
```bash
vim /etc/openstack-dashboard/local_settings.py

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

### memcached 세션 저장소 구성
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'


### openstack 호스트가 controller 임을 명시
OPENSTACK_HOST = "controller"

### identity Service API의 URL 명시(버전 3 사용)
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST


TIME_ZONE = "Asia/Seoul"

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_NEUTRON_NETWORK = {
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

### keystone Multidomain 지원 설정
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True


### 기본 설정으로 사용할 도메인 설정(대시보드를 통해 사용자 생성 시 기본값으로 사용됨)
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
```
#### (3) /etc/apache2/conf-available/openstack-dashboard.conf 파일에서 기본적으로 필요한 WSGIApplicationGroup %{GLOBAL} 구문이 빠져있다면 추가한다.
```
WSGIApplicationGroup %{GLOBAL}
```

#### (4) 서비스 재시작
```
systemctl restart apache2 memcached
```

#### (5) 결과 확인

#### - [admin 사용자의 ID, Password는 Keystone bootstrap에서 설정했다](https://github.com/revenge1005/OpenStack-Install-and-build/tree/main/2.%20OpenStack%20-%20Keystone%20%EC%84%A4%EC%B9%98%20%EB%B0%8F%20%EC%84%A4%EC%A0%95#1-4-%ED%99%98%EA%B2%BD-%EB%B3%80%EC%88%98-%EC%84%A4%EC%A0%95)

