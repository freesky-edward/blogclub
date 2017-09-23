# Running the kubernetes on OpenStack step by step

标签： kubernetes

---

## Introduction

OpenStack is an open source software for creating private and public clouds. The main purpose of OpenStack is providing the Infrastructure as a Service (IaaS). There are a collection of projects in OpenStack that as a whole provide a complete IaaS solution. 

Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications. The nodes(instances) provided by OpenStack Nova can be used for running the kubernetes. And the volumes provided by OpenStack Cinder also can be used by the pods in the kubernetes. 

This guide will take you through the steps of deploying Kubernetes to OpenStack manually. Firstly we will install a devstack environment, and then create a Nova instance in this devstack. We will compile and install a single node kubernetes environment in this Nova instance.

## Install a devstack environment

DevStack is a series of extensible scripts used to quickly bring up a complete OpenStack environment based on the latest versions of everything from git master. It is used interactively as a development environment and as the basis for much of the OpenStack project’s functional testing. So here we will use devstack to install a complete OpenStack environment for running the kubernetes.


### 1 The environment

OS: Ubuntu 16.04 
CPU: Intel(R) Xeon(R) CPU E5-2407 0 @ 2.20GHz
Memory: 32G
Storage: 100G

### 2 Download DevStack
Devstack should be run as a non-root user with sudo enabled (standard logins to cloud images such as "ubuntu" or "cloud-user" are usually fine).
You can quickly create a separate stack user to run DevStack with:
```bash
$ sudo useradd -s /bin/bash -d /opt/stack -m stack
```
Since this user will be making many changes to your system, it should have sudo privileges:
```bash
$ echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
$ sudo su - stack
```
Download DevStack:
```bash
$ git clone https://git.openstack.org/openstack-dev/devstack
$ cd devstack
```
The devstack repo contains a script that installs OpenStack and templates for configuration files.

### 3 Create a local.conf and Install 
```bash
[[local|localrc]]

DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_PASSWORD=password
SERVICE_TOKEN=password
ADMIN_PASSWORD=password

enable_plugin karbor http://git.trystack.cn/openstack/karbor master
enable_plugin karbor-dashboard http://git.trystack.cn/openstack/karbor-dashboard master
enable_plugin heat http://git.trystack.cn/openstack/heat master

#run the services you want to use
ENABLED_SERVICES=rabbit,mysql,key
ENABLED_SERVICES+=,n-cpu,n-api,n-obj,n-cond,n-sch,n-novnc,n-cauth,n-api-meta
ENABLED_SERVICES+=,placement-api,placement-client
ENABLED_SERVICES+=,neutron,q-svc,q-agt,q-dhcp,q-l3,q-meta,q-metering
ENABLED_SERVICES+=,cinder,g-api,g-reg
ENABLED_SERVICES+=,c-api,c-vol,c-sch,c-bak,horizon
ENABLED_SERVICES+=,heat,h-api,h-api-cfn,h-api-cw,h-eng

#Add the karbor services
enable_service karbor-api
enable_service karbor-operationengine
enable_service karbor-protection

#Add the karbor-dashboard services
enable_service karbor-dashboard

#disable the default services you don't want to use
disable_service n-net
disable_service n-cell

SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5
SWIFT_REPLICAS=1
SWIFT_DATA_DIR=$DEST/data
enable_service s-proxy s-object s-container s-account


```
The project karbor is a data protection orchestration as a service in OpenStack. It will be installed in this devstack. 

The OpenStack Cloud Provider will be used in this kubernetes environment. In this usecase, the n-api-meta and q-meta services must be enabled. The kubernetes in nova instance will use the nova metadata api to get the detail information about this instance. This detail information will be used in the initialization phase of the openstack cloud provider when the kubernetes service is starting.

**The Nova Metadata Service**
OpenStack provides a metadata service for cloud instances. This metadata is useful for accessing instance-specific information from within the instance. The primary purpose of this capability is to apply customizations to the instance during boot time if cloud-init or cloudbase-init is configured on your Linux or Windows image, respectively. However, instance metadata can be accessed at any time after the instance boots by the user or by applications running on the instance. 

How to get the metadata information via the nova Metadata Service in the instance:
```bash
#curl http://169.254.169.254/openstack/latest 
meta_data.json
```

Start the install:
```bash
./stack.sh
```

###  4 Configure the Network
There are lots of special requirements(hardware, software, network, performance, etc) about the environment for the kubernetes running on the openstack nova instance. The network of this nova instance must be able to access the public network. We need the public network to install the single node kubernetes and its dependencies in this instance. 
To see more detail information about how to configure the Neutron network for the virtual machines accessing the public network, we can read this article: [The configuration of OpenStack Neutron](http://www.cnblogs.com/edisonxiang/p/7365627.html).

Import envionment variables for OpenStack clients:
```bash
root@ubuntu:/opt# source adminrc.sh 
root@ubuntu:/opt# cat adminrc.sh 
#!/bin/sh

export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_TENANT_NAME=admin
export OS_AUTH_URL="http://192.168.98.122/identity"
export OS_IDENTITY_API_VERSION=3
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default

```
The Neutron network configuration:
```bash
root@ubuntu:/opt# neutron net-list
+--------------------------------------+-------------+----------------------------------+------------------------------------------------------+
| id                                   | name        | tenant_id                        | subnets                                              |
+--------------------------------------+-------------+----------------------------------+------------------------------------------------------+
| 712107d0-3a6f-4dc6-934f-d9e85d0b1e01 | privatesite | 5004e1415225483681f672cb8a8b01a2 | bea6e1df-ba18-4879-93f3-1caf09c61555 10.0.0.0/24     |
| aaae5111-ce68-4540-851c-48368adddad9 | publicsite  | 5004e1415225483681f672cb8a8b01a2 | b2079351-efdc-4630-81db-fa6b31187e53 192.168.98.0/24 |
+--------------------------------------+-------------+----------------------------------+------------------------------------------------------+
root@ubuntu:/opt# neutron net-show aaae5111-ce68-4540-851c-48368adddad9
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2017-08-17T02:05:36Z                 |
| description               |                                      |
| id                        | aaae5111-ce68-4540-851c-48368adddad9 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| is_default                | False                                |
| mtu                       | 1500                                 |
| name                      | publicsite                           |
| port_security_enabled     | True                                 |
| project_id                | 5004e1415225483681f672cb8a8b01a2     |
| provider:network_type     | flat                                 |
| provider:physical_network | ext                                  |
| provider:segmentation_id  |                                      |
| revision_number           | 5                                    |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   | b2079351-efdc-4630-81db-fa6b31187e53 |
| tags                      |                                      |
| tenant_id                 | 5004e1415225483681f672cb8a8b01a2     |
| updated_at                | 2017-08-25T08:35:40Z                 |
+---------------------------+--------------------------------------+
```

The Neutron sub-network configuration:
```bash
root@ubuntu:/opt# neutron subnet-list
+--------------------------------------+--------------------+----------------------------------+-----------------+----------------------------------------------------+
| id                                   | name               | tenant_id                        | cidr            | allocation_pools                                   |
+--------------------------------------+--------------------+----------------------------------+-----------------+----------------------------------------------------+
| b2079351-efdc-4630-81db-fa6b31187e53 | publicsite-subnet  | 5004e1415225483681f672cb8a8b01a2 | 192.168.98.0/24 | {"start": "192.168.98.30", "end": "192.168.98.40"} |
| bea6e1df-ba18-4879-93f3-1caf09c61555 | privatesite-subnet | 5004e1415225483681f672cb8a8b01a2 | 10.0.0.0/24     | {"start": "10.0.0.50", "end": "10.0.0.100"}        |
+--------------------------------------+--------------------+----------------------------------+-----------------+----------------------------------------------------+
root@ubuntu:/opt# neutron subnet-show b2079351-efdc-4630-81db-fa6b31187e53
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.98.30", "end": "192.168.98.40"} |
| cidr              | 192.168.98.0/24                                    |
| created_at        | 2017-08-17T02:05:38Z                               |
| description       |                                                    |
| dns_nameservers   | 218.6.200.139                                      |
| enable_dhcp       | True                                               |
| gateway_ip        | 192.168.98.1                                       |
| host_routes       |                                                    |
| id                | b2079351-efdc-4630-81db-fa6b31187e53               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | publicsite-subnet                                  |
| network_id        | aaae5111-ce68-4540-851c-48368adddad9               |
| project_id        | 5004e1415225483681f672cb8a8b01a2                   |
| revision_number   | 1                                                  |
| service_types     |                                                    |
| subnetpool_id     |                                                    |
| tags              |                                                    |
| tenant_id         | 5004e1415225483681f672cb8a8b01a2                   |
| updated_at        | 2017-08-25T08:35:40Z                               |
+-------------------+----------------------------------------------------+

```
###  5 Creating a Nova instance for running the single node kubernetes
This instance needs more than 4GB of memory for compiling the source code of the kubernetes.

Create the flavor for the instance:
```bash
root@ubuntu:/opt# nova flavor-create m1.k8s 10 4096 10 2 
+----------------------------+--------+
| Property                   | Value  |
+----------------------------+--------+
| OS-FLV-DISABLED:disabled   | False  |
| OS-FLV-EXT-DATA:ephemeral  | 0      |
| disk                       | 10     |
| extra_specs                | {}     |
| id                         | 10     |
| name                       | m1.k8s |
| os-flavor-access:is_public | True   |
| ram                        | 4096   |
| rxtx_factor                | 1.0    |
| swap                       |        |
| vcpus                      | 2      |
+----------------------------+--------+
root@ubuntu:/opt# 
```
Upload the [Ubuntu 16.04 LTS cloud image](https://cloud-images.ubuntu.com/xenial/current/) to Glance:
```bash
root@ubuntu:/opt# glance image-create --file ./xenial-server-cloudimg-amd64-disk1.img  --name ubuntu16_password
root@ubuntu:/opt# glance image-list
+--------------------------------------+--------------------------+
| ID                                   | Name                     |
+--------------------------------------+--------------------------+
| 95a8298a-65ef-48b7-828e-164b39bf9f3e | cirros-0.3.5-x86_64-disk |
| e73eb245-97da-4519-b026-1f75ef9cab80 | ubuntu16_password        |
+--------------------------------------+--------------------------+
```


Create the instance:
```bash
nova boot --flavor  m1.k8s --image e73eb245-97da-4519-b026-1f75ef9cab80  --access-ip-v4 aaae5111-ce68-4540-851c-48368adddad9 k8s-freesky
+--------------------------------------+----------------------------------------------------------+
| Property                             | Value                                                    |
+--------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                    | AUTO                                                     |
| OS-EXT-AZ:availability_zone          | nova                                                     |
| OS-EXT-SRV-ATTR:host                 | ubuntu                                                   |
| OS-EXT-SRV-ATTR:hostname             | k8s-freesky                                              |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | ubuntu                                                   |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000007                                        |
| OS-EXT-SRV-ATTR:kernel_id            |                                                          |
| OS-EXT-SRV-ATTR:launch_index         | 0                                                        |
| OS-EXT-SRV-ATTR:ramdisk_id           |                                                          |
| OS-EXT-SRV-ATTR:reservation_id       | r-34tfwper                                               |
| OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                 |
| OS-EXT-SRV-ATTR:user_data            | -                                                        |
| OS-EXT-STS:power_state               | 1                                                        |
| OS-EXT-STS:task_state                | -                                                        |
| OS-EXT-STS:vm_state                  | active                                                   |
| OS-SRV-USG:launched_at               | 2017-08-25T07:45:40.000000                               |
| OS-SRV-USG:terminated_at             | -                                                        |
| accessIPv4                           |                                                          |
| accessIPv6                           |                                                          |
| config_drive                         |                                                          |
| created                              | 2017-08-25T07:45:31Z                                     |
| description                          | k8s-freesky                                              |
| flavor:disk                          | 10                                                       |
| flavor:ephemeral                     | 0                                                        |
| flavor:extra_specs                   | {}                                                       |
| flavor:original_name                 | m1.k8s                                                   |
| flavor:ram                           | 4096                                                     |
| flavor:swap                          | 0                                                        |
| flavor:vcpus                         | 2                                                        |
| hostId                               | c8a939831baddec6d107181907ca9b8839c80eb268ab11aa7b8129a7 |
| host_status                          | UP                                                       |
| id                                   | 87c7e359-ba4b-412d-b7e5-4fe482f33012                     |
| image                                | ubuntu16_password (e73eb245-97da-4519-b026-1f75ef9cab80) |
| key_name                             | -                                                        |
| locked                               | False                                                    |
| metadata                             | {}                                                       |
| name                                 | k8s-freesky                                              |
| os-extended-volumes:volumes_attached | []                                                       |
| progress                             | 0                                                        |
| publicsite network                   | 192.168.98.40                                            |
| security_groups                      | default                                                  |
| status                               | ACTIVE                                                   |
| tags                                 | []                                                       |
| tenant_id                            | 5004e1415225483681f672cb8a8b01a2                         |
| updated                              | 2017-08-25T07:45:40Z                                     |
| user_id                              | 32d23d2bab774c4d8a061b3c83a8a171                         |
+--------------------------------------+----------------------------------------------------------+
```
Generate a route table about the ip address 169.254.169.254:
```bash
root@ubuntu:/opt/kube# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.98.122  0.0.0.0         UG    0      0        0 ens3
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.98.0    *               255.255.255.0   U     0      0        0 ens3
root@ubuntu:/opt/kube# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:85:0e:87:c7  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ens3      Link encap:Ethernet  HWaddr fa:16:3e:cc:95:3c  
          inet addr:192.168.98.40  Bcast:192.168.98.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fecc:953c/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:4116103 errors:0 dropped:0 overruns:0 frame:0
          TX packets:235816 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1194455376 (1.1 GB)  TX bytes:23193366 (23.1 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:160 errors:0 dropped:0 overruns:0 frame:0
          TX packets:160 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:11840 (11.8 KB)  TX bytes:11840 (11.8 KB)

root@ubuntu:/opt/kube# dhclient ens3
root@ubuntu:/opt/kube# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.98.122  0.0.0.0         UG    0      0        0 ens3
169.254.169.254 192.168.98.30   255.255.255.255 UGH   0      0        0 ens3
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.98.0    *               255.255.255.0   U     0      0        0 ens3
root@ubuntu:/opt/kube# 

```

Login this instance and check the nova metadata service:
```bash
root@ubuntu:/opt/stack/karbor# ssh ubuntu@publicsite=192.168.98.40
ubuntu@ubuntu:~$ curl 169.254.169.254/1.0/meta-data
ami-id
ami-launch-index
ami-manifest-path
hostname
instance-id
local-ipv4
reservation-id
security-groups
ubuntu@ubuntu:~$
```

## Install a single node kubernetes on Nova instance

Requirements:

OS: Ubuntu 16.04.2 LTS
CPU: 2 vCPUs
Memory: 4G
Storage: 10G

Docker:  1.12.6
Go: 1.9
Etcd: v3.2.6
Kubernetes:master

###  1 Install tools:
Install git: 
```bash
sudo apt-get install git
```

Install gcc: 
```bash
sudo apt-get install gcc
```

###  2 Install docker:
```bash
root@ubuntu:/#apt-get update
root@ubuntu:/#apt-get upgrade

root@ubuntu:/#apt-get install docker.io
root@ubuntu:/# docker --version
Docker version 1.12.6, build 78d1802
```

###  2 Install go:
Use the default 1.8.3 version of go to compile the master branch of the kubernetes, we will get an error about 'out of memory'. So we will install a 1.9 version of go in our Nova instance.

Now download the Go language binary archive file using the following link. To find and download latest version available or 64 bit version go to the official [download page](https://golang.org/dl/).
```bash
root@ubuntu:/# cd /opt/
root@ubuntu:/opt# wget https://storage.googleapis.com/golang/go1.9rc2.linux-amd64.tar.gz
--2017-09-22 12:42:26--  https://storage.googleapis.com/golang/go1.9rc2.linux-amd64.tar.gz
Resolving storage.googleapis.com (storage.googleapis.com)... 172.217.27.144
Connecting to storage.googleapis.com (storage.googleapis.com)|172.217.27.144|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 98420498 (94M) [application/x-gzip]
Saving to: 'go1.9rc2.linux-amd64.tar.gz'

go1.9rc2.linux-amd64.tar.gz                                100%[========================================================================================================================================>]  93.86M  10.2MB/s    in 8.1s    

2017-09-22 12:42:35 (11.6 MB/s) - 'go1.9rc2.linux-amd64.tar.gz' saved [98420498/98420498]
```

Now extract the downloaded archive and install it to the desired location on the system. For this tutorial, I am installing it under /usr/local directory. You can also put this under the home directory (for shared hosting) or other location.
```bash
root@ubuntu:/opt# sudo tar -xvf  go1.9rc2.linux-amd64.tar.gz 

```
Now we need to setup Go language environment variables for your project. Commonly you need to set 3 environment variables as GOROOT, GOPATH and PATH.
GOROOT is the location where Go package is installed on your system.
```bash
vi ~/.bashrc

export GOROOT=/opt/go
export GOPATH=/opt/kube/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```

###  3 Install etcd:
```bash
root@ubuntu:/opt/kube# wget  https://storage.googleapis.com/etcd/v3.2.6/etcd-v3.2.6-linux-amd64.tar.gz

root@ubuntu:/opt/kube# tar xzvf etcd-v3.2.6-linux-amd64.tar.gz 

root@ubuntu:/opt/kube# mv etcd-v3.2.6-linux-amd64 etcd
```

setup etcd environment variables
```bash
vi ~/.bashrc

export GOROOT=/opt/go
export GOPATH=/opt/kube/go
export PATH=/opt/kube/etcd:$GOPATH/bin:$GOROOT/bin:$PATH

root@ubuntu:/opt/kube# etcd --version
etcd Version: 3.2.6
Git SHA: 9d43462
Go Version: go1.8.3
Go OS/Arch: linux/amd64
```

###  3 Install and compile the kubernetes:
```bash
root@ubuntu:/opt/kube# git clone https://github.com/kubernetes/kubernetes.git

root@ubuntu:/opt/kube#  make all WHAT=cmd/hyperkube
root@ubuntu:/opt/kube#  make all WHAT=cmd/kubectl
```

Install and setup cfssl:
```bash
root@ubuntu:/opt/kube/go/bin# go get github.com/cloudflare/cfssl/cmd/...
root@ubuntu:/opt/kube/go/bin# go install github.com/cloudflare/cfssl/cmd/...

root@ubuntu:/opt/kube/go/bin# ll
total 62484
drwxr-xr-x 2 root root     4096 Aug 25 07:49 ./
drwxr-xr-x 5 root root     4096 Aug 25 07:47 ../
-rwxr-xr-x 1 root root 17006184 Aug 25 07:49 cfssl*
-rwxr-xr-x 1 root root  6819862 Aug 25 07:47 cfssl-bundle*
-rwxr-xr-x 1 root root  6366730 Aug 25 07:47 cfssl-certinfo*
-rwxr-xr-x 1 root root  7318988 Aug 25 07:47 cfssl-newkey*
-rwxr-xr-x 1 root root  7611517 Aug 25 07:47 cfssl-scan*
-rwxr-xr-x 1 root root  2379049 Aug 25 07:47 cfssljson*
-rwxr-xr-x 1 root root  6156997 Aug 25 07:47 mkbundle*
-rwxr-xr-x 1 root root 10299920 Aug 25 07:47 multirootca*

```

###  4 configure and run the kubernetes:

setup cloud provider environment variables
```bash
root@ubuntu:/opt/kube/go/bin# vi ~/.bashrc 

export GOROOT=/opt/go
export GOPATH=/opt/kube/go
export PATH=/opt/kube/etcd:$GOPATH/bin:$GOROOT/bin:/opt/kube/kubernetes/cluster:$PATH
export HOSTNAME_OVERRIDE=k8s-chenying
export KUBELET_HOST=0.0.0.0
export CLOUD_PROVIDER=openstack
export CLOUD_CONFIG=/opt/config/cloud_config.conf

vi  /opt/config/cloud_config.conf
[Global]
auth-url="http://192.168.98.122/identity/v3"
username="admin"
password="password"
region="RegionOne"
tenant-id="5004e1415225483681f672cb8a8b01a2"
domain-name="Default"

[BlockStorage]
bs-version="v2"
```

The tenant-id about admin and the region name in cloud_config.conf are got form OpenStack Keystone:
```bash
(openstack) project list
+----------------------------------+--------------------+
| ID                               | Name               |
+----------------------------------+--------------------+
| 0734daab6e1f4bda85565b7a4b749654 | swiftprojecttest1  |
| 3979e378062f4821875a07dd36d7bbec | alt_demo           |
| 4be080fe0a8949bb8cca50ac121c6500 | swiftprojecttest4  |
| 5004e1415225483681f672cb8a8b01a2 | admin              |
| 5ea25fe4b55f495d949dd4aa1aba1289 | service            |
| 8e66caa0f20344d1a5152dfcc869ad09 | swiftprojecttest2  |
| b640b765a8a3465d873bd86cb7ceb919 | demo               |
| facba6d3218848749dfb499ef94cbf76 | invisible_to_admin |
+----------------------------------+--------------------+

(openstack) endpoint list
+----------------------------------+-----------+--------------+----------------+---------+-----------+------------------------------------------------------+
| ID                               | Region    | Service Name | Service Type   | Enabled | Interface | URL                                                  |
+----------------------------------+-----------+--------------+----------------+---------+-----------+------------------------------------------------------+
| 0b9f8360cfa248ae897fbd2bc27dc74d | RegionOne | karbor       | data-protect   | True    | public    | http://192.168.98.122/data-protect/v1/$(project_id)s |
| 10a6552a6bee42efbb58ba5a08cd73cb | RegionOne | karbor       | data-protect   | True    | internal  | http://192.168.98.122/data-protect/v1/$(project_id)s |
| 19de7a6e82b040dea6bd4c9b9edcf591 | RegionOne | keystone     | identity       | True    | public    | http://192.168.98.122/identity                       |
| 1f83f2e5a1e740a0ad7e4f8b3f783265 | RegionOne | neutron      | network        | True    | public    | http://192.168.98.122:9696/                          |
| 2126f13c03464e5688236b4bab6b90eb | RegionOne | swift        | object-store   | True    | public    | http://192.168.98.122:8080/v1/AUTH_$(project_id)s    |
| 28c8600490494991af9056d0ed7e056e | RegionOne | heat         | orchestration  | True    | admin     | http://192.168.98.122/heat-api/v1/$(project_id)s     |
| 307a5fd0bee643cf88ee037fae5fc5a3 | RegionOne | keystone     | identity       | True    | admin     | http://192.168.98.122/identity                       |
| 3b384c9e684c4c75b7097476ad2347de | RegionOne | swift        | object-store   | True    | admin     | http://192.168.98.122:8080                           |
| 5034fdb4ff804c658cd1bbf53dbd360b | RegionOne | nova_legacy  | compute_legacy | True    | public    | http://192.168.98.122/compute/v2/$(project_id)s      |
| 55a073e3a7f34728843a425d9fbdd449 | RegionOne | heat-cfn     | cloudformation | True    | internal  | http://192.168.98.122/heat-api-cfn/v1                |
| 560c201dfc2844d48eadbdf31ee3668e | RegionOne | heat-cfn     | cloudformation | True    | public    | http://192.168.98.122/heat-api-cfn/v1                |
| 6a2556cf02714024b7d3e0b8b418b3e1 | RegionOne | karbor       | data-protect   | True    | admin     | http://192.168.98.122/data-protect/v1/$(project_id)s |
| 8b9f2469d3bf4c1780fc07d10a626d03 | RegionOne | glance       | image          | True    | public    | http://192.168.98.122/image                          |
| 94b47a3a629549cd8befb4f261a42f55 | RegionOne | cinderv3     | volumev3       | True    | public    | http://192.168.98.122/volume/v3/$(project_id)s       |
| b09cc56d620f4da98dc44f1c2504e23a | RegionOne | heat-cfn     | cloudformation | True    | admin     | http://192.168.98.122/heat-api-cfn/v1                |
| bbbd39e0496042f39cc90663329fef15 | RegionOne | heat         | orchestration  | True    | public    | http://192.168.98.122/heat-api/v1/$(project_id)s     |
| bcb89d4d1ad64bcaa1c09868d0645170 | RegionOne | cinderv2     | volumev2       | True    | public    | http://192.168.98.122/volume/v2/$(project_id)s       |
| be5dfec88f444d2fb997002b5097d74b | RegionOne | nova         | compute        | True    | public    | http://192.168.98.122/compute/v2.1                   |
| c7cbebe4fbba48dd84bf63c9867723d8 | RegionOne | cinder       | volume         | True    | public    | http://192.168.98.122/volume/v1/$(project_id)s       |
| ea347475790d4daebe0d179f77111896 | RegionOne | heat         | orchestration  | True    | internal  | http://192.168.98.122/heat-api/v1/$(project_id)s     |
| ed21320dd4194ccfb3498c56133f7994 | RegionOne | placement    | placement      | True    | public    | http://192.168.98.122/placement                      |
+----------------------------------+-----------+--------------+----------------+---------+-----------+------------------------------------------------------+

```

Run the single node kubernettes:
```bash
root@ubuntu:/opt/kube/kubernetes/hack# ./local-up-cluster.sh -O

```

Test creating a pod:

```bash

export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig


root@ubuntu:/opt/yaml# cat pod-cinder.yaml
apiVersion: v1
kind: Pod
metadata:
    name: busybox-test
    labels:
        name: busybox-test
spec:
    restartPolicy: Never
    containers:
    - resources:
        limits:
          cpu: 0.5
      image: gcr.io/google_containers/busybox
      command:
         - "/bin/sh"
         - "-c"
         - "while true; do date >> /mnt/test/pod-out.txt; sleep 1; done"
      name: busybox

root@ubuntu:/opt/yaml# kubectl.sh create -f pod-cinder.yaml

root@ubuntu:/opt/yaml# kubectl.sh get pod
NAME                                                       READY     STATUS    RESTARTS   AGE
busybox-test                                               1/1       Running   0          0d
```


















