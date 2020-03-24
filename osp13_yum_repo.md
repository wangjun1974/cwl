
获取OSP13软件仓库步骤

1. 在可联网的环境里安装RHEL7.5操作系统

2. 
注册节点
```
subscription-manager register
```

查找可用订阅
```
subscription-manager list --available --all --matches="Red Hat OpenStack"
```

附加可用订阅
```
subscription-manager attach --pool=Valid-Pool-Number-123456
```

启用软件仓库
```
subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-13-rpms --enable=rhel-7-server-rhceph-3-tools-rpms
```

3. 执行以下脚本同步软件仓库内容到本地

```
#!/bin/bash

mkdir -p /repos
localPath="/repos"
fileConn="/getPackage/"


for i in rhel-7-server-rpms rhel-7-server-extras-rpms rhel-7-server-rh-common-rpms rhel-ha-for-rhel-7-server-rpms rhel-7-server-rhceph-3-tools-rpms rhel-7-server-openstack-13-rpms rhel-7-server-openstack-13-tools-rpms rhel-7-server-openstack-13-devtools-rpms rhel-7-server-openstack-13-optools-rpms 
do

  mkdir -p "$localPath"$i"$fileConn"
  rm -rf "$localPath"$i"$fileConn"repodata

done

  echo "sync channel $i..."
  time reposync --plugins --newest-only --delete --download_path="$localPath" --source --repoid=rhel-7-server-rpms --repoid=rhel-7-server-extras-rpms  --repoid=rhel-7-server-rh-common-rpms  --repoid=rhel-ha-for-rhel-7-server-rpms  --repoid=rhel-7-server-rhceph-3-tools-rpms --repoid=rhel-7-server-openstack-13-rpms --repoid=rhel-7-server-openstack-13-tools-rpms --repoid=rhel-7-server-openstack-13-devtools-rpms --repoid=rhel-7-server-openstack-13-optools-rpms

for i in rhel-7-server-rpms rhel-7-server-extras-rpms rhel-7-server-rh-common-rpms rhel-ha-for-rhel-7-server-rpms rhel-7-server-rhceph-3-tools-rpms rhel-7-server-openstack-13-rpms rhel-7-server-openstack-13-tools-rpms rhel-7-server-openstack-13-devtools-rpms rhel-7-server-openstack-13-optools-rpms
do

  echo "create repo $i..."
  time createrepo --update --skip-stat --cachedir /tmp/empty-cache-dir "$localPath"$i

done

exit 0
```

脚本执行完后，osp13对应软件仓库将同步在/repo目录下
