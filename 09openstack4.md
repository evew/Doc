# OpenStack 4

### 网络服务配置(Networking Service -Neutron)

部署节点 controller Node

在mariadb中创建neutron数据库

    mysql -u root -p

    create database neutron;

    grant all privileges on neutron.* to 'neutron'@'localhost' identified by 'NEUTRON_DBPASS';
    grant all privileges on neutron.* to 'neutron'@'%' identified by 'NEUTRON_DBPASS';

创建网络服务证书和API路径

    source ~/.openstack/.admin-openrc

    openstack user create --domain default --password-prompt neutron

将admin 角色授予neutron

    openstack role add -- project service --user neutron admin

创建neutron 服务实体

    openstack service create --name neutron --description "OpenStack Networking" network

创建网络服务API路径

    openstack endpoint create --region RegionOne network public http://svctl1:9696
    openstack endpoint create --region RegionOne network internal http://svctl1:9696
    openstack endpoint create --region RegionOne network admin http://svctl1:9696

安装配置neutron-server 服务组件

    sudo apt-get install neutron-server neutron-plugin-ml2

修改配置文件 

    sudo vim /etc/neutron/neutron.conf

注释 connection = sqlite:...

    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@svctl1/neutron 
    
    [DEFAULT]
    core_plugin = ml2
    service_plugins = router
    allow_overelapping_ips = True
    rpc_backend = rabbit
    auth_strategy = keystone
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True

    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = RABBIT_PASS
    
    [keystone_authtoken]
    auth_uri = http://svctl1:5000
    auth_url = htto://svctl1:35357
    memcached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    
    [nova]
    auth_url = http://svctl1:35357
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = nova
    password = NOVA_PASS

配置Modular layer 2 (ML2) 插件

    sudo vim /etc/neutron/plubins/ml2/ml2_conf.ini

    [ml2]
    type_drivers = flat,vlan,vxlan
    tenant_network_types = vxlan
    mechanism_drivers = linuxbridge,l2population
    extension_drivers = port_security

    [ml2_type_flat]
    flat_networks = provider

    [ml2_type_vxlan]
    vni_ranges = 1:1000

    [securitygroup]
    enable_ipset = True

将配置信息写入neutron 数据库

    su root
    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

为计算服务配置网络访问服务

    sudo vim /etc/nova/nova.conf

    [neutron]
    url = http://svctl1:9696
    auth_url = http://svctl1:35357
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS

    service metadata_proxy = True
    metadata_proxy_shared_secret = METADATA_SECRET

### 部署节点Network Node

在Network 节点上部署组件：neutron-linuxbridge-agnet,neutron-l3-agnet, neutron-dhcp-agent, neutron-metadata-agent

    sudo apt-get install neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

配置公共服务组件

    sudo vi /etc/neutron/neutron.conf

    [DEFAULT]
    rpc_backend - rabbit
    auth_strategy = keystone

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
    username = neutron
    password = NEUTRON_PASS


配置linux 网桥代理

    sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini

    [linux_bridge]
    physical_interface_mappings = provider:enp3s0f0
    
    [vxlan]
    enable_vxlan = True
    local_ip = 10.0.0.21
    l2_population = True

    [securitygroup]
    enable_security_grop = true
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver


配置3层网络代理

    sudo vim /etc/neutron/l3_agent.ini
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
    external_network_bridge =
    

配置dhcp代理

    sudo vim /etc/neutron/dhcp_agent.ini

    
