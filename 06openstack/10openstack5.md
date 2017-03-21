# OpenStack Install

### 仪表盘服务配置(Dashboard Service - Horizon)

部署节点：controller node

安装horizon

    sudo apt-get install openstack-dashboard

    sudo dpkg --remove --force-remove-reinstreq openstack-dashboard-ubuntu-theme

    sudo vim /etc/openstack-dashboard/local_settings.py

    OPENSTACK_HOST = "svctl1"

    ALLOWED_HOSTS = ['*', ]

配置memcached session storage service 注释掉其他session storage配置信息

    SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache'
            'LOCATION': 'svctl1:11211',
            }
        }

    
    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
    OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPOORT = True

    OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume":2,
        }
    OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"

    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

    TIME_ZONE = "Asia/Shanghai"

重启Apache

    sudo service apache2 reload
