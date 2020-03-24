osp13 离线 registry 准备

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

3. 执行以下命令

创建stack用户并且设置口令
```
useradd stack
passwd stack
```

设置stack用sudoer权限
```
echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack
```

切换为stack用户
```
su - stack
```

创建images和templates目录
```
mkdir ~/images
mkdir ~/templates
```

设置主机名
```
sudo hostnamectl set-hostname undercloud.example.com
sudo hostnamectl set-hostname --transient undercloud.example.com
```

为/etc/hosts文件添加undercloud记录
```
sudo -i 
cat >> /etc/hosts << 'EOF'
10.66.208.233 undercloud.example.com undercloud
EOF
```

从stack用户退回到root用户
```
exit
```

更新系统并重启
```
sudo yum update -y
sudo reboot
```

重启后，使用stack用户登陆系统，安装director
```
sudo yum install -y python-tripleoclient
```

创建undercloud.conf文件
```
cat > ~/undercloud.conf << 'EOF'
[DEFAULT]
enable_tempest = false
local_interface = eth1
[auth]
undercloud_admin_password = Redhat01
[ctlplane-subnet]
EOF
```

安装director
```
openstack undercloud install
```

4. 
注意：因为系统需要下载镜像，请事先安装docker并启动docker服务
保存以下内容，并且执行准备images脚本
```
#!/bin/bash
if [ $PWD != /home/stack ] ; then echo "USAGE: $0 this script needs to be executed in /home/stack"; exit 1 ; fi

DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
template_base_dir="$DIR"

source /home/stack/stackrc

openstack overcloud container image prepare \
  --namespace=registry.access.redhat.com/rhosp13 \
  --push-destination=192.168.24.1:8787 \
  --prefix=openstack- \
  -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
  --set ceph_namespace=registry.access.redhat.com/rhceph \
  --set ceph_image=rhceph-3-rhel7 \
  --tag-from-label {version}-{release} \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/barbican.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/cinder-backup.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/collectd.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/congress.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/ec2-api.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/etcd.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/fluentd.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/ironic-inspector.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/ironic.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/manila.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/mistral.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/neutron-ovn-dvr-ha.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/neutron-ovn-dvr.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/neutron-ovn-ha.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/neutron-ovn.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/octavia.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/sensu-client.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/zaqar.yaml \
  --output-env-file ${template_base_dir}/overcloud_images.yaml \
  --output-images-file ${template_base_dir}/local_registry_images.yaml
```

假设输出env文件保存在~/templates/local_registry_images.yaml中

生成下载镜像脚本
```
cat templates/local_registry_images | grep imagename | sort -u | sed -e 's,- imagename: ,docker pull ,' | tee /tmp/download-images.sh
```

下载镜像
```
sh -x /tmp/download-images.sh
```

打包镜像到其他服务器保存
```
tar zcvf - /var/lib/docker | ssh <user@remote> "cat > /home/images/backup/var-lib-docker-$(date -I).tar.gz"
```

5. 
在内网undercloud上，创建并且清空registry
```
systemctl start docker
docker rmi $(docker images -q)
systemctl stop docker
```

在备份服务器上远程使用备份恢复内网undercloud的/var/lib/docker
```
cat < /home/images/backup/var-lib-docker-2018-09-20.tar.gz | ssh root@undercloud "cd /; tar zxvf -"
```

重新启动内网undercloud docker服务
```
systemctl start docker
```

在内网undercloud上重新打tag，注意：192.168.24.1需用内网undercloud的ip地址替换
```
docker images | grep registry | sed -e 's,registry.access.redhat.com,192.168.24.1:8787,' | awk '{print "docker tag "$3" "$1":"$2"; docker push "$1":"$2 }' | tee /tmp/tag-and-push.sh 

sh -x /tmp/tag-and-push.sh
```

