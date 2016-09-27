# OpenStack 3

### 部署节点 Compute Node

在compute节点上需要安装nova-compute

### 安装配置计算服务组件nova-compute

    sudo apt-get install nova-compute
    
修改配置文件

    sudo vim /etc/nova/nova.conf

    [DEFAULT]
    rpc_backend = rabbit
    auth_stategy = keystone
    my_ip = 10.0.0.31
    use_neutron = True
    firewall_driver = nova.virt.firewall.NoopFirewallDriver

    [oslo_messaging_rabbit]
    rabbit_host = svctl1
    rabbit_userid = openstack
    rabbit_password = RABBIT_PASS

    [keystone_authtoken]
    auth_usi = http://svctl1:5000
    auth_url = http://svctl1:35357
    memcached_servers = svctl1:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = nova
    password = NOVA_PASS

    [vnc]
    enabled = True
    vncserver_listen = 0.0.0.0
    vncserver_proxyclient_address = $my_ip
    novncproxy_base_url = http://svctl1:6080/vnc_auto.html

    [glance]
    api_servers = http://svctl1:9292

    [oslo_concurrency]
    lock_path = /var/lib/nova/tmp

删除[DEFAULT]处的logdir
    
    logdir=/var/log/nova

重启计算服务

    sudo service nova-compute restart

在controller节点上验证计算服务

    source ~/.openstack/.admin-openrc
    openstack compute service list


