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