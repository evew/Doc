# OpenStack 2

### 镜像服务配置(Image Service - Glance)

部署节点，Controller Node
在MariaDB中创建glance数据库

    mysql -u root -p

    create database glance;

    grant all privileges on glance.* to 'glance'@'localhost' identified by 'GLANCE_DBPASS';
    grant all privileges on glance.* to 'glance'@'%' identified by 'GLANCE_DBPASS';


### 创建glance 服务实体和API路径

    source ~/.openstack/.admin-openrc

    openstack user create --domain default --password-prompt glance


将admin角色授予glance用户和service 计划

    openstack role add --project service --user glance admin

创建glance服务实体

    openstack service create --name glance --description "OpenStack Image" image

创建镜像服务API路径

    openstack endpoint create --region RegionOne image public http://svctl1:9292
    openstack endpoint create --region RegionOne image internal http://svctl1:9292
    openstack endpoint create --region RegionOne image admin http://svctl1:9292

安装和配置glance 服务组件

    sudo apt-get install glance

    sudo vim /etc/glance/glance-api.conf

    connection = mysql+pymysql://glance:GLANCE_DBPASS@svctl1/glance

    [keystone_authtoken]
    auth_uri = http://svctl1:5000
    auth_url = http://svctl1:35357
    memcached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = glance
    password = GLANCE_PASS

    [paste_deploy]
    flavor = keystone

    [glance_store]
    stores = file,http
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/


修改配置文件 

    sudo vim /etc/glance/glance-registry.conf

    connection = mysql+pymysql://glance:GLANCE_DBPASS@svctl1/glance

    [keystone_authtoken]
    auth_uri = http://svctl1:5000
    auth_url = http://svctl1:35357
    mecached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = glance
    password = GLANCE_PASS

    [paste_deploy]
    flavor = keystone

将配置信息写入glance数据库

    su root
    su -s /bin/sh -c "glance-manage db_sync" glance

    sudo service glance_registry restart
    sudo service glance_api restart

验证glance服务组件配置

    source ~/.openstack/.admin-openrc

    wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

    openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public


# 计算服务配置(Compute Service - Nova)

### 部署节点：Controller Node
在Controller 节点上需要安装nova-api、nova-conductor、nova-consoleauth、nova-novncproxy、nova-scheduler

在mariadb 中创建nova_api 和 nova

    mysql -u root -p

    create database nova_api;
    create database nova;

    grant all privileges on nova_api.* to 'nova'@'localhost' identified by 'NOVA_DBPASS';
    grant all privileges on nova_api.* to 'nova'@'%' identified by 'NOVA_DBPASS';
    grant all privileges on nova.* to 'nova'@'localhost' identified by 'NOVA_DBPASS';
    grant all privileges on nova.* to 'nova'@'%' identified by 'NOVA_DBPASS';

创建计算服务证书和API路径

    source ~/.openstack/.admin-openrc

创建nova用户

    openstack user create --domain default --password-prompt nova

将admin角色授予nova

    openstack role add -- project service --user nova admin

创建nova服务实体

    openstack service create --name nova --description "OpenStack Compute" compute

创建计算服务API路径

    openstack endpoint create --region RegionOne compute public http://svctl1:8774/v2.1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne compute internal http://svctl1:8774/v2.1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne compute admin http://svctl1:8774/v2.1/%\(tenant_id\)s

安装nova服务组件

    sudo apt-get install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler

    sudo vim /etc/nova/nova.conf
