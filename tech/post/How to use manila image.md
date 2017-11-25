# How to use manila image in devstack pike version

The environment is pike release openstack that built by devstack [1].

## 1. Add the manila image to glance
We can get manila-service-image.qcow2 from [tarballs](https://tarballs.openstack.org/manila-image-elements/images/)
or build image from manila-image-elements code
glance image-create --name "manila-service-image" --file manila-service-image.qcow2 --disk-format qcow2 --container-format bare --visibility public

## 2. Create an instance by image
nova flavor-list
nova boot --flavor d1   --image  f4bb472c-2e52-49e5-a6b5-41a3ff8c7b86  zjinstance

## 3. Check the status of the instance

root@instance-4:/opt/stack/tempest# nova list

+--------------------------------------+------------+--------+------------+-------------+--------------------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks                       |
+--------------------------------------+------------+--------+------------+-------------+--------------------------------+
| 1b2f1d4e-6503-4660-923d-fdce5b7d9d2e | zjinstance | ACTIVE | -          | Running     | public=2001:db8::3, 172.24.4.5 |
+--------------------------------------+------------+--------+------------+-------------+--------------------------------+

Then I run "ssh manila@172.24.4.5", and try to login the instance, but it doesn't work.


## 4. So I login instance from console link, and check if the ip is configured in instance.
After login instance and run "ifconfig", I find this isn't have "172.24.4.5" in instance.
root@instance-4:/opt/stack/tempest# nova get-vnc-console  1b2f1d4e-6503-4660-923d-fdce5b7d9d2e novnc

+-------+---------------------------------------------------------------------------------+
| Type  | Url                                                                             |
+-------+---------------------------------------------------------------------------------+
| novnc | http://10.140.0.4:6080/vnc_auto.html?token=62baf4b9-9f3a-4352-8979-49edf75cc3cd |
+-------+---------------------------------------------------------------------------------+

note::

root@instance-4:neutron subnet-show  public

If the "enable_dhcp" is set equal to false，the system can not automatic create a ip for instance.
So we need to config ip for instance.


## 5.  Add 172.24.4.5 ip into instance.

$ sudo ifonfig ens3 172.24.4.5 up


Then I run "ssh manila@172.24.4.5" try to login the instance, but it still doesn't work

## 6. Check wheather we config the seucrity-group for instance
root@instance-4:/opt/stack/tempest# neutron port-list | grep 4.5
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.

| 0e903be6-31e5-4ed8-9d40-7bd146de6b55 |      | e87f0f23d8994de8b317d62764733fff | fa:16:3e:3a:34:15 | {"subnet_id": "ec455e1c-b75a-4b09-b327-cbe84d7a8b6d", "ip_address": "fd83:177a:6d37::1"}                    |
| 171f08c9-70ad-4bdf-bbbf-d12be2aa4abe |      | e87f0f23d8994de8b317d62764733fff | fa:16:3e:48:e7:fa | {"subnet_id": "8eafdb1f-7f24-438a-9fe5-d485f655d0bf", "ip_address": "10.0.0.2"}                             |
|                                      |      |                                  |                   | {"subnet_id": "ec455e1c-b75a-4b09-b327-cbe84d7a8b6d", "ip_address": "fd83:177a:6d37:0:f816:3eff:fe48:e7fa"} |
| 8b48a8b0-edc0-4d5a-959c-edf35794c743 |      | 4901985e873b4984af8d22a60f7b15f3 | fa:16:3e:7c:6e:fc | {"subnet_id": "0b540976-801a-45db-b7ae-a5565ca653f4", "ip_address": "10.2.5.8"}                             |
| 93577ead-d78b-429b-bfac-f5b49eb8d456 |      | 6edab66a84074e449b37d2692a52e6e1 | fa:16:3e:54:38:a0 | {"subnet_id": "df2f2e93-4117-4377-9b79-6edbd6de18bf", "ip_address": "172.24.4.5"}                           |
|                                      |      |                                  |                   | {"subnet_id": "506db691-2f18-49c6-9e66-d9af63844854", "ip_address": "2001:db8::3"}                          |
|                                      |      |                                  |                   | {"subnet_id": "506db691-2f18-49c6-9e66-d9af63844854", "ip_address": "2001:db8::1"}                          |
| b635ef7b-8d6d-4056-a8de-f73b59b7f087 |      | 4901985e873b4984af8d22a60f7b15f3 | fa:16:3e:77:2c:ca | {"subnet_id": "bebaa066-d1b6-4e92-9e31-77f9ceda2279", "ip_address": "10.254.0.18"}                          |
|                                      |      |                                  |                   | {"subnet_id": "f4f5d9b7-068c-48bb-b6c8-30583753d200", "ip_address": "10.254.0.2"}                           |
| c8d6c7a6-5aa9-440a-9902-bfdf4550a88f |      | e87f0f23d8994de8b317d62764733fff | fa:16:3e:8d:de:d3 | {"subnet_id": "8eafdb1f-7f24-438a-9fe5-d485f655d0bf", "ip_address": "10.0.0.1"}                             |
| f14ccd40-e571-4daf-957c-8a00fc534f33 |      | 4901985e873b4984af8d22a60f7b15f3 | fa:16:3e:d2:8b:02 | {"subnet_id": "f4f5d9b7-068c-48bb-b6c8-30583753d200", "ip_address": "10.254.0.11"}                          |

root@instance-4:/opt/stack/tempest# neutron port-show 93577ead-d78b-429b-bfac-f5b49eb8d456
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.

+-----------------------+------------------------------------------------------------------------------------+
| Field                 | Value                                                                              |
+-----------------------+------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                               |
| allowed_address_pairs |                                                                                    |
| binding:host_id       | instance-4                                                                         |
| binding:profile       | {}                                                                                 |
| binding:vif_details   | {"port_filter": true, "datapath_type": "system", "ovs_hybrid_plug": true}          |
| binding:vif_type      | ovs                                                                                |
| binding:vnic_type     | normal                                                                             |
| created_at            | 2017-09-26T07:01:00Z                                                               |
| description           |                                                                                    |
| device_id             | 1b2f1d4e-6503-4660-923d-fdce5b7d9d2e                                               |
| device_owner          | compute:nova                                                                       |
| extra_dhcp_opts       |                                                                                    |
| fixed_ips             | {"subnet_id": "df2f2e93-4117-4377-9b79-6edbd6de18bf", "ip_address": "172.24.4.5"}  |
|                       | {"subnet_id": "506db691-2f18-49c6-9e66-d9af63844854", "ip_address": "2001:db8::3"} |
| id                    | 93577ead-d78b-429b-bfac-f5b49eb8d456                                               |
| mac_address           | fa:16:3e:54:38:a0                                                                  |
| name                  |                                                                                    |
| network_id            | 25f6981b-431b-480d-81ce-2909cb9140dc                                               |
| port_security_enabled | True                                                                               |
| project_id            | 6edab66a84074e449b37d2692a52e6e1                                                   |
| revision_number       | 7                                                                                  |
| security_groups       | a5a6d1b7-7762-44bd-ba6c-e3d719e4d02e                                               |
| status                | ACTIVE                                                                             |
| tags                  |                                                                                    |
| tenant_id             | 6edab66a84074e449b37d2692a52e6e1                                                   |
| updated_at            | 2017-09-26T07:01:31Z                                                               |
+-----------------------+------------------------------------------------------------------------------------+

## 7.  Add seucrity-group 

root@instance-4:/opt/stack/tempest# neutron security-group-rule-create a5a6d1b7-7762-44bd-ba6c-e3d719e4d02e --protocol tcp --port-range-min 22 --port-range-max 22 --direction ingress
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Created a new security_group_rule:

+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2017-09-26T08:12:36Z                 |
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 8bb05fd2-23d6-4259-b899-f367590a7cbb |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 6edab66a84074e449b37d2692a52e6e1     |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_prefix  |                                      |
| revision_number   | 0                                    |
| security_group_id | a5a6d1b7-7762-44bd-ba6c-e3d719e4d02e |
| tenant_id         | 6edab66a84074e449b37d2692a52e6e1     |
| updated_at        | 2017-09-26T08:12:36Z                 |
+-------------------+--------------------------------------+

## 8. It is ok, we can login our instance by ssh.
root@instance-4: ssh manila@172.24.4.5
The authenticity of host '172.24.4.5 (172.24.4.5)' can't be established.
ECDSA key fingerprint is SHA256:bHDGJnCnW+MvZGXKagJNXAa3X4bMmNM8ssNRp10QPpA.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.24.4.5' (ECDSA) to the list of known hosts.
manila@172.24.4.5's password: 
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-96-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Tue Sep 26 08:06:18 2017
manila@ubuntu:~$ 

## 9.  You can use Manila Shared filesystem from an Instance as follows.
manila@ubuntu:~$ sudo mkdir /test

We can mount a share in the instance after create a share [2].
manila@ubuntu:~$ sudo mount -t nfs 172.24.4.13:/shares/share-368f94c9-9538-4329-a07a-706c8e2743b6 /test 


[1]  https://docs.openstack.org/devstack/latest/
[2]  https://docs.openstack.org/project-install-guide/shared-file-systems/draft/post-install.html

