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



