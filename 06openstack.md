# Openstack Ubuntu 16.04


    sudo passwd root

    su -
    sudo vi /etc/hosts
    sudo apt-get update
    sudo apt-get dis-upgrade

### 安装时间服务

    sudo apt-get install chrony

    sudo vim /etc/chrony/chrony.conf

    allow 10.0.0.0/24
    server cn.pool.ntp.org iburst
    server 127.127.1.0 iburst

    sudo service chrony restart


### 安装openstack client:

    sudo apt-get install software-properties-common
    sudo apt-get update
    sudo apt-get dist-upgrade
    sudo apt-get install python-openstackclient

### 安装mariadb

    sudo apt-get install mariadb-server python-mysqldb
    sudo mysqladmin -u root password Database_PASS
    sudo vim /etc/mysql/mariadb.conf.d/openstack.cnf

    [mysqld]
    bind-address = 10.0.0.11
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    character-set-server = utf8

    sudo service mysql restart
    sudo service mysql status
    
    sudo mysql_secure_installation

    mysql -uroot -p
    mysql -h localhost -uroot -p

    sudo mysql -u root
    use mysql;
    update user set plugin='' where User='root';
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Database_PASS';
    flush privileges;
    
### 安装mongodb

    sudo apt-get install mongodb-server mongodb-clients python-pymongo

    sudo vim /etc/mongodb.conf

    bind_ip = 10.0.0.11
    smallfiles = true


### 安装RabbitMQ 消息队列服务

    sudo apt-get install rabbitmq-server

    sudo rabbitmqctl add_user openstack RABBIT_PASS

    sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"


### 安装memcached

    sudo apt-get install memcached python-memcache

    sudo vim /etc/memcached.conf

    -l 10.0.0.11

    sudo service memcached restart

到此controller node 的 基础服务已经装好了。

### 身份服务配置 (Identity service - Keystone)

Identity 服务采用RESTful设计，使用REST API 提供 Web服务接口

在mariadb 中创建Keystone 数据库

    mysql -u root -p

    create database keystone;
    grant all privileges on keystone.* to 'keystone'@'localhost' identified by 'KEYSTONE_PASS'
    grant all privileges on keystone.* to 'keystone'@'%' identified by 'KEYSTONE_PASS'

    mysql -hlocalhost -ukeystone -p

生成临时管理身份认证令牌 (ADMIN_TOKEN)
生成一个随机值，作为keystone初始配置时的ADMIN_TOKEN

    openssl rend -hex 10


安装keystone 和apache http server with mod_wsgi
采用apache http server with mod_wsgi 监听端口5000和35357 提供身份服务。默认keystone服务已经监听端口5000和35357,为了避免冲突，首先关闭keystone 服务。

    sudo vim /etc/init/keystone.override

添加内容：

    manual

安装keystone 和apache http server

    sudo apt-get install keystone apache2 libapache2-mod-wsgi

修改keystone配置文件

    sudo vim /etc/keystone/keystone.conf

    admin_token = ADMIN_TOKEN

    connection = mysql+pymysql://heystone:KEYSTONE_DBPASS@svctl1/keystone

    provider = fernet

    su root
    su -s /bin/sh -c "keystone-manage db_sync" keystone

    sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone


配置apache http server

    sudo vim /etc/apache2/apache2.conf

    ServerName svctl1

新建文件

    sudo vim /etc/apache2/sites-available/wsgi-keystone.conf

    Listen 5000
    Listen 35357

    <VirtualHost *:5000>
        WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone
        WSGIProcessGroup keystone-public
        WSGIScriptALIAS / /urs/bin/keystone-wsgi-public
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        ErrorLogFormat "%{cu}t %M"
        ErrorLog /var/log/apache2/keystone.log
        CustomLog /var/log/apache2/keystone_access.log combined

        <Directory /usr/bin>
            Require all granted
        </Directory>
    </VirtualHost>

    <VirtualHost *:35357>
        WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone
        WSGIProcessGroup keystone-admin
        WSGIScriptALIAS / /urs/bin/keystone-wsgi-admin
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        ErrorLogFormat "%{cu}t %M"
        ErrorLog /var/log/apache2/keystone.log
        CustomLog /var/log/apache2/keystone_access.log combined

        <Directory /usr/bin>
            Require all granted
        </Directory>
    </VirtualHost>

启用身份服务虚拟主机

    sudo ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled

    sudo service apache2 restart

删除keystone配置信息默认数据库

    sudo rm -f /var/lib/keystone/keystone.db
    sudo service keystone restart

创建服务实体(Service Entity) 和API路径(API Endpoints)
向openstack命令传递认证令牌值和身份服务URL

    export OS_TOKEN=ADMIN_TOKEN
    export OS_URL=http://svctl1:35357/v3
    export OS_IDENTITY API_VERSION=3

创建服务实体

    openstack service create --name keystone --description "OpenStack Identity" identity

创建API路径

    openstack endpoint create --region RegionOne identity public http://svctl1:5000/v3
    openstack endpoint create --region RegionOne identity internal http://svctl1:5000/v3
    openstack endpoint create --region RegionOne identity admin http://svctl1:35357/v3

创建域、计划、用户、角色

    openstack domain create --description "Default Domain" default
    openstack project create --domain default --description "Admin Project" admin
    openstack user create --domain default --password-prompt admin
    openstack role create admin

将admin角色授予admin 计划和admin用户

    openstack role add --project admin --user admin admin

创建服务计划

    openstack project create --domain default --description "Service Project" service

创建示例计划、示例用户、普通用户角色

    openstack project create --domain default --description "Demo Project" demo
    openstack user create --domain default --password-prompt demo
    openstack role create user

将普通用户角色授予示例计划和示例用户

    openstack role add --project demo --user demo user


### 验证keystone 组件配置

出于安全考虑，禁用临时身份认证令牌机制
修改文件

    sudo vim /etc/keystone/keystone-paste.ini

    删除[pipeline:public_api] 等 admin_token_auth 共3处

取消环境变量

    unset OS_TOKEN OS_URL

为admin申请一个身份认证令牌

    openstack --os-auth-url http://svctl1:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue

为demo申请一个身份认证令牌

    openstack --os-auth-url http://svctl1:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue

为admin创建openstack 脚本

    vim ~/.openstack/.admin-openrc

    export OS_PROJECT_DOMAIN_NAME=default
    export OS_USER_DOMAIN_NAME=default
    export OS_PROJECT_NAME=admin
    export OS_USERNAME=admin
    export OS_PASSWORD=ADMIN_PASS
    export OS_AUTH_URL=http://svctl1:35357/v3
    export OS_AUTH_TYPE=password
    export OS_IDDENTITY_API_VERSION=3
    export OS_IMAGE_API_VERSION=2  



