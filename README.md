# Readhat Openstack Platform 13

Readhat Openstack Platform 13 is openstack product by redhat

##  Underclod

 Interface on VM Undercloud

|  Interface     |        ROLE          |
|----------------|----------------------|
|Public          |`External`            |
|IPMI            |`Management Hardware` |
|PXE_BOOT        |`For Boot OS`         |


Install Director [DOC](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/installing-the-undercloud) on OS RHEL 7.9. This Sample Config undercloud.conf

```bash
[DEFAULT]
undercloud_hostname = lab.example.com
overcloud_domain_name = example.com
local_ip = 10.127.1.181/24
undercloud_public_host = [ ip public]
undercloud_admin_host = 10.127.1.182
undercloud_nameservers = 8.8.8.8
undercloud_ntp_servers = [ NTP SERVER ] 
generate_service_certificate = false
local_interface = eth2
undercloud_debug = false
enable_tempest = true
local_subnet = ctlplane-subnet


[ctlplane-subnet]
cidr = 10.127.1.0/24
dhcp_start = 10.127.1.183
dhcp_end = 10.127.1.200
inspection_iprange = 10.127.1.240,10.127.1.250
gateway = 10.127.1.181
masquerade = true

```

Run the following command to install the director on the undercloud:
```sh
openstack undercloud install
```

## OVERCLOUD
1. Config Local Registry
```sh
#  gen file config image
openstack overcloud container image prepare \
  --namespace=registry.access.redhat.com/rhosp13 \
  --push-destination=[local-ip]:8787 \
  --prefix=openstack- \
  --tag-from-label {version}-{release} \
  --output-env-file=/home/stack/templates/overcloud_images.yaml \
  --output-images-file /home/stack/local_registry_images.yaml
```
```sh
# push image to local registry
sudo openstack overcloud container image upload --config-file  /home/stack/local_registry_images.yaml --verbose  
```
```sh
# verify
curl http://local-ip:8787/v2/_catalog | jq .repositories[]
```

2. Import Node
```sh
openstack overcloud node import allnode.json
```
```sh
# default config allnode.json
{
  "nodes": [
    {
        "mac":[
            "cc:cc:cc:cc:cc:cc"
        ],
        "pm_type":"ipmi",
        "pm_user":"ADMIN",
        "pm_password":"ADMIN",
        "pm_addr":"1.1.1.1",
        "name": "controller-0"
    },
    {
        "mac":[
            "cc:cc:cc:cc:cc:cc"
        ],
        "pm_type":"ipmi",
        "pm_user":"ADMIN",
        "pm_password":"ADMIN",
        "pm_addr":"1.1.1.1",
        "name": "compute-0"
    }
 ]
}
```
```sh
# verify node
openstack baremetal node list
```
3. Introspect Node
```sh
openstack overcloud node introspect --all-manageable --provide
or 
openstack baremetal node manage [NODE UUID]
openstack overcloud node introspect [NODE UUID] --provide
```

4. Add Profile to Node
```sh
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' [NODE UUID]
```
```sh
openstack overcloud profiles list
```

5. Generate File Config Overcloud

```sh
# roles 
openstack overcloud roles generate -o ~/templates/custom-templates/01-roles_data.yaml Controller Compute
```
```sh
# network
cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml ~/templates/custom-templates/02-network_data.yaml
```
```sh
# node info
vi /home/stack/templates/node-info.yaml
""""
parameter_defaults:
  OvercloudControllerFlavor: control
  OvercloudComputeFlavor: compute
  OvercloudCephStorageFlavor: ceph-storage
  ControllerCount: 3
  ComputeCount: 3
  CephStorageCount: 3
"""
```
```sh
# render enviroment
mkdir ~/rendered
cd /usr/share/openstack-tripleo-heat-templates/
tools/process-templates.py -r ~/templates/custom-templates/01-roles_data.yaml -n ~/templates/custom-templates/02-network_data.yaml -o ~/rendered/
```
```sh
# Config Network
cp rendered/environments/network-environment.yaml ~/templates/custom-templates/06-network-environment.yaml
cp rendered/environments/network-isolation.yaml ~/templates/custom-templates/05-network-isolation.yaml
cp rendered/environments/net-single-nic-with-vlans.yaml ~/templates/custom-templates/07-net-single-nic-with-vlans.yaml
cp rendered/environments/ips-from-pool-all.yaml ~/templates/custom-templates/08-ips-from-pool-all.yaml
cp rendered/environments/fixed-ip-vips.yaml ~/templates/custom-templates/09-fixed-ip-vips.yaml
```
6. Script Deploy overcloud
```bash
#!/bin/bash

START_TIME=$(date)
echo $START_TIME

source /home/stack/stackrc

THT=/usr/share/openstack-tripleo-heat-templates

time openstack overcloud deploy \
  --templates /usr/share/openstack-tripleo-heat-templates/ \
  --timeout 240 \
  -r /home/stack/nice-overcloud/templates/custom-templates/01-roles_data.yaml \
  -n /home/stack/nice-overcloud/templates/custom-templates/02-network_data.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/03-node_info.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/04-overcloud_images.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/05-network-isolation.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/06-net-single-nic-with-vlans.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/07-network-environment.yaml \
  -e /home/stack/nice-overcloud/templates/custom-templates/08-ips-from-pool-all.yaml \
  --debug --log-file /home/stack/nice-overcloud/logs/overcloudDeploy.log

END_TIME=$(date)
echo $START_TIME
echo $END_TIME
```
```sh
nohub bash -x deploy-overcloud.sh &
```

## Command 
```sh
# destroy overcloud
openstack stack delete overcloud --wait --yes 
swift delete --all
```
```sh
# watch process deploy
watch -n 60 "openstack stack list --nested -c 'Stack Name' -c 'Stack Status' | grep -v _COMPLETE"
```

