# Openstack Install

### 块存储服务配置(Block Storage Service - Cinder)

部署节点：Controller Node
创建cinder数据库

    mysql -u root -p
    create database cinder;
    
    grant all privileges on cinder.* to 'cinder'@'localhost' identified by 'CINDER_DBPASS';
    grant all privileges on cinder.* to 'cinder'@'%' identified by 'CINDER_DBPASS'


创建cinder服务实体和API路径

    source ~/.openstack/.admin_openrc

在openstack 中创建一个cinder用户

    openstack user create --domain default --password-prompt cinder

将admin角色授予cinder用户

    openstack role add --project service --user cinder admin

创建cinder 和 cinder2 服务实体
    
    openstack service create --name cinder --description "OpenStack Block Storage" volume
    openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2

创建块存储服务API路径

    openstack endpoint create --region RegionOne volume public http://svctl1:8776/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne volume internal http://svctl1:8776/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne volume admin http://svctl1:8776/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne volumev2 public http://svctl1:8776/v2/%\(tenant_id\)s
    openstack endpoint create --region RegionOne volumev2 internal http://svctl1:8776/v2/%\(tenant_id\)s
    openstack endpoint create --region RegionOne volumev2 admin http://svctl1:8776/v2/%\(tenant_id\)s

安装和配置cinder服务组件

    sudo apt-get install cinder-api cinder-scheduler

修改配置文件

    sudo vim /etc/cinder/cinder.conf

    [database]
    connection = mysql+pymysql://cinder:CINDER_DBPASS@svctl1/cinder

    [DEFAULT]
    rpc_backend = rabbit
    auth_strategy = keystone
    my_ip = 10.0.0.11

    [oslo_messaging_rabbit]
    rabbit_host = svctl1
    rabbit_userid = openstack
    rabbit_password = RABBIT_PASS
    
    [keystone_authtoken]
    auth_uri = http://svctl1:5000
    auth_url = http://svctl1:35357
    memcached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = cinder
    password = CINDER_PASS

    [oslo_concurrency]
    lock_path = /var/lib/cinder/tmp

将配置信息写入块存储服务数据库cinder

    su root
    su -s /bin/sh -c "cinder-manage db sync" cinder

配置计算服务调用块存储服务

    sudo vim /etc/nova/nova.conf

    [cinder]
    os_region_name = RegionOne

重启计算服务API和块存储服务

    sudo service nova-api restart
    sudo service cinder-scheduler restart
    sudo service cinder-api restart

部署节点:BlockStorage Node

安装配置LVM:
    
    sudo apt-get install lvm2

创建LVM物理卷/dev/sdb:

    sudo pvcreate /dev/sdb

创建LVM卷组cinder-volumes

    sudo vgcreate cinder-volumes /dev/sdb

配置只有OpenStack 实例才可以访问块存储卷

    sudo vim /etc/lvm/lvm.conf

    filter = [ "a/sdb1/", "r/.*/"]

安装cinder块存储服务组件

    sudo apt-get install cinder-volume

修改配置

    sudo vim /etc/cinder/cinder.conf

    [database]
    connection = mysql+pymysql://cinder:CINDER_DBPASS@svctl1/cinder

    [DEFAULT]
    rpc_backend = rabbit
    auth_strategy = keystone
    my_ip = 10.0.0.51
    enabled_backends = lvm
    glance_api_servers = http://svctl1:9292

    [oslo_messaging_rabbit]
    rabbit_host = svctl1
    rabbit_userid = openstack
    rabbit_password = RABBIT_PASS

    [keystone_authtoken]
    auth_uri = http://svctl1:5000
    auth_url = http://svctl1:35357
    memcached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = cinder
    password = CINDER_PASS
    
    [lvm]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    iscsi_protocol = iscsi
    iscsi_helper = tgtadm

    [oslo_concurrency]
    lock_path = /var/lib/cinder/tmp

重启块存储服务

    sudo service tgt restart
    sudo service cinder-volume restart

验证块存储服务

    source ~/.opnestack/.admin-openrc

    cinder service-list
