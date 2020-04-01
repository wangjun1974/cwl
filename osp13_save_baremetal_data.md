### 保存baremetal node信息

```
#!/bin/bash
source ~/stackrc

mkdir -p ~/ironic
pushd ~/ironic

for node in $(openstack baremetal node list --format value --field name)
do 
  openstack baremetal introspection data save $node > inspector_data-$node
done

popd
```