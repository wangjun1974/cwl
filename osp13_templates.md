## 模版修改记录

### 部署脚本
原来的部署脚本 deploy.sh
```
#!/bin/bash

set +x
openstack overcloud deploy \
 --templates \
 -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.j2.yaml \
 -e templates/network-environment.yaml \
 -e templates/base.yaml \
 --control-flavor control \
 --compute-flavor compute \
 --neutron-bridge-mappings tenant:br-tenant,datacenter:br0 \
 --neutron-network-vlan-ranges tenant:2001:2048
```

改完的部署脚本 deploy.sh
```
#!/bin/bash

set +x
openstack --debug overcloud deploy --templates \
  -p /usr/share/openstack-tripleo-heat-templates/plan-samples/plan-environment-derived-params.yaml \
  -e /home/stack/templates/node-info.yaml \
  -e /home/stack/templates/overcloud_images.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
  -e /home/stack/templates/network_environment.yaml \
  -e /home/stack/templates/base.yaml \
  --ntp-server <time_server_ip>\
  --log-file /tmp/overcloud_deploy_`date +%Y%m%d_%H%M%S`.log
```

修改说明：
* 采用默认plan和network isolation方式进行网络隔离
```
  -p /usr/share/openstack-tripleo-heat-templates/plan-samples/plan-environment-derived-params.yaml
...
  -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml
```

* 节点类型及数量在/home/stack/templates/node-info.yaml中定义

* 使用以下文件指定部署受用的镜像
```
  -e /home/stack/templates/overcloud_images.yaml
```

* 指定网络设置
```
  -e /home/stack/templates/network_environment.yaml
```

* 指定ntp服务器
```
  --ntp-server <time_server_ip>
```

### templates/node-info.yaml
注意⚠️：新增
通过这个文件指定部署节点及数量
```
parameter_defaults:                  
  OvercloudControllerFlavor: control
  OvercloudComputeFlavor: compute
  ControllerCount: 3
  ComputeCount: 2
```

### templates/base.yaml
注意⚠️：新增
通过这个文件指定时区及其他存储相关设置
```
parameter_defaults:
  TimeZone: 'Asia/Shanghai'
  CinderEnableIscsiBackend: false
  CinderEnableRbdBackend: false
  CinderEnableNfsBackend: false
  NovaEnableRbdBackend: false
  GlanceBackend: swift
  GnocchiBackend: swift
  SwiftReplicas: 3  
```


### templates/network-environment.yaml

修改前的内容
```
resource_registry:
{%- for role in roles %}
  OS::TripleO::{{role.name}}::Net::SoftwareConfig: ../network/config/single-nic-linux-bridge-vlans/{{role.deprecated_nic_config_name|default(role.name.lower() ~ ".yaml")}}
{%- endfor %}
resource_registry:
  OS::TripleO::Compute::Net::SoftwareConfig: nic-configs/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig: nic-configs/controller.yaml
  OS::TripleO::CephStorage::Net::SoftwareConfig: nic-configs/ceph-storage.yaml
parameter_defaults:
  TenantNetCidr: 172.168.2.0/24
  TenantAllocationPools: [{'start': '172.168.2.10', 'end': '172.168.2.200'}]
  TenantNetworkVlanID: 1003
  InternalApiNetCidr: 172.168.1.0/24
  InternalApiAllocationPools: [{'start': '172.168.1.10', 'end': '172.168.1.200'}]
  InternalApiNetworkVlanID: 1002
  StorageNetCidr: 172.168.3.0/24
  StorageAllocationPools: [{'start': '172.168.3.10', 'end': '172.168.3.200'}]
  StorageNetworkVlanID: 1004
  StorageMgmtNetCidr: 172.168.4.0/24
  StorageMgmtAllocationPools: [{'start': '172.168.4.10', 'end': '172.168.4.200'}]
  StorageMgmtNetworkVlanID: 1005
  ExternalNetCidr: 172.23.101.0/22
  ExternalAllocationPools: [{'start': '172.23.101.2', 'end': '172.23.101.6'}]
  ExternalInterfaceDefaultRoute: 172.23.100.254
       
  ControlPlaneDefaultRoute: 172.168.0.1       
  EC2MetadataIp: 172.168.0.1
                        
  NeutronExternalNetworkBridge: "br0"
```

修改后的内容
```
resource_registry:
  OS::TripleO::Compute::Net::SoftwareConfig: nic-configs/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig: nic-configs/controller.yaml
parameter_defaults:
  TenantNetCidr: 172.168.2.0/24
  TenantAllocationPools: [{'start': '172.168.2.10', 'end': '172.168.2.200'}]
  TenantNetworkVlanID: 1003
  InternalApiNetCidr: 172.168.1.0/24
  InternalApiAllocationPools: [{'start': '172.168.1.10', 'end': '172.168.1.200'}]
  InternalApiNetworkVlanID: 1002
  StorageNetCidr: 172.168.3.0/24
  StorageAllocationPools: [{'start': '172.168.3.10', 'end': '172.168.3.200'}]
  StorageNetworkVlanID: 1004
  StorageMgmtNetCidr: 172.168.4.0/24
  StorageMgmtAllocationPools: [{'start': '172.168.4.10', 'end': '172.168.4.200'}]
  StorageMgmtNetworkVlanID: 1005
  ExternalNetCidr: 172.23.101.0/22
  ExternalAllocationPools: [{'start': '172.23.101.2', 'end': '172.23.101.6'}]
  ExternalNetworkVlanID:1312
  ExternalInterfaceDefaultRoute: 172.23.100.254
       
  ControlPlaneDefaultRoute: 172.168.0.1
  EC2MetadataIp: 172.168.0.1

  NeutronNetworkType: 'vxlan,vlan'
  NeutronTunnelTypes: 'vxlan'
  NeutronNetworkVLANRanges: 'datacentre:1312:1312'
  NeutronBridgeMappings: 'datacentre:br-ex'
  NeutronEnableIsolatedMetadata: True
  
  NetworkDeploymentActions: ['CREATE','UPDATE']
```

修改说明：
* 不使用ceph，因此移除OS::TripleO::CephStorage::Net::SoftwareConfig

* External Network Vlan ID是1312
```
  ExternalNetworkVlanID:1312
```

* overcloud同时支持vxlan和vlan租户网络
* 默认创建vlan网络datacentre
* 指定datacentre vlan网络id范围
* 通过neutron提供metadata服务
```
  NeutronNetworkType: 'vxlan,vlan'
  NeutronTunnelTypes: 'vxlan'
  NeutronNetworkVLANRanges: 'datacentre:1312:1312'
  NeutronBridgeMappings: 'datacentre:br-ex'
  NeutronEnableIsolatedMetadata: True
```

* 指定部署或更新都可配置网络
```
  NetworkDeploymentActions: ['CREATE','UPDATE']
```

### 获取baremetal节点信息
参考: 