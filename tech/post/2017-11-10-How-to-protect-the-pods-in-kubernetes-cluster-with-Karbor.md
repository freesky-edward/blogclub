# How to protect the pods in kubernetes cluster with Karbor


---

## Introduction

With the rapid development of cloud computing, there is a trend of explosive growth in cloud data over recent years. Cloud data backup and recovery has become an urgent topic which the customers concern. Running Kubernetes on Openstack becames more and more popular. The data protection of the application on Kubernetes also need be considered.

In this article we would like to introduce a way, how to use Karbor to protect your application deployed in the pod of the Kubernetes which runs on top of OpenStack. The application data protected by Karbor include the configurations and metadata in etcd service, and the persistent volume provided by Cinder.


## Karbor
Karbor is a data proection orchestration as a service in OpenStack. To protect the Data and Metadata that comprises an OpenStack-deployed Application against loss/damage (e.g. backup, replication) by providing a standard framework of APIs and services that allows vendors to provide plugins through a unified interface.

[More information about Karbor](https://wiki.openstack.org/wiki/karbor)

![karbor-logo](https://wiki.openstack.org/w/images/e/e7/OpenStack_Project_Karbor.png)

The karbor provides a plug-in mechanism to integrate and orchestrate existing vendor and open source resources protection solutions. The kubernetes cluster can run on openstack instances using Openstack cloud provider, the pods can be created with persistent volumes provided by Cinder.

When we want to get the pods from the kubernetes in Karbor, we need define a new resoure type 'OS::Kubernetes::Pod'. We need develop a new protectable plugin for this resource type. The volumes provided by Cinder can be mounted to the pod. We can say the mounted volumes are the sub-resources of this pod in Karbor. We can define the dependency between the pod and volume protectables in the default distribution of Karbor. 

When we want to protect the pod of the kubernetes, we need develop a new protection plugin for the reouse type 'OS::Kubernetes::Pod'. In this plugin, the configurations and metadata in etcd service about the pods can be backuped and restored. We have the protection plugins about the volume in Karbor already. We don't need develop a new plugin to protect the mounted volumes of the pod.

When the pod with mounted volumes being protected, the workflow engine of karbor will create a taskflow of the pod first. The workflow engine will resolve the dependency resouces of this pod. The protection taskflow about the mounted volumes also will be created automatically. As a result, the pod metadata and the mounted volumes will be protected at the same time in one checkpoint.


## The pod protectable plugin in Karbor

The protectable plugin is responsible for the implementation of getting a type of protectable
element which Karbor can protect. Most prominently OpenStack resources (volume, project, server, pod, etc). The actual instance of protectable element is named Resource. The Protectable Plugin about one type resource defines what types resource it depends on. It defines the dependency between
different resource type in the default distribution of Karbor.

The Karbor code-base has a python API that corresponds to the set of API calls
you must implement to be a Karbor protectable plugin. Within the source code directory,
look at karbor/service/protection/protectable_plugin.py

A protectable plugin must implement the following methods:

### 1 get_resource_type: 
This function will return the resource type that this plugin supports.

```python
    _SUPPORT_RESOURCE_TYPE = constants.POD_RESOURCE_TYPE
    def get_resource_type(self):
        return self._SUPPORT_RESOURCE_TYPE
```

### 2 get_parent_resource_types: 
This function will return the possible parent resource types.
```python
    def get_parent_resource_types(self):
        return (constants.PROJECT_RESOURCE_TYPE)
```

### 3. list_resources: 
This function will list resource instances of type this plugin supported.
```python
    def list_resources(self, context, parameters=None):
        try:
            pods = self._client(context).list_namespaced_pod(self.namespace)
        except Exception as e:
            LOG.exception("List all summary pods from kubernetes failed.")
            raise exception.ListProtectableResourceFailed(
                type=self._SUPPORT_RESOURCE_TYPE,
                reason=six.text_type(e))
        else:
            return [resource.Resource(
                type=self._SUPPORT_RESOURCE_TYPE,
                id=uuid.uuid5(uuid.NAMESPACE_OID, "%s:%s" % (
                    self.namespace, pod.metadata.name)),
                name="%s:%s" % (self.namespace, pod.metadata.name),
                extra_info={'namespace': self.namespace})
                for pod in pods.items
                if pod.status.phase not in INVALID_POD_STATUS]
```

### 4. show_resource: 
This function will show one resource detail information.
```python
    def show_resource(self, context, resource_id, parameters=None):
        try:
            if not parameters:
                raise
            name = parameters.get("name", None)
            if ":" in name:
                pod_namespace, pod_name = name.split(":")
            else:
                pod_namespace = self.namespace
                pod_name = name
            pod = self._client(context).read_namespaced_pod(
                pod_name, pod_namespace)
        except Exception as e:
            LOG.exception("Show a summary pod from kubernetes failed.")
            raise exception.ProtectableResourceNotFound(
                id=resource_id,
                type=self._SUPPORT_RESOURCE_TYPE,
                reason=six.text_type(e))
        else:
            if pod.status.phase in INVALID_POD_STATUS:
                raise exception.ProtectableResourceInvalidStatus(
                    id=resource_id, type=self._SUPPORT_RESOURCE_TYPE,
                    status=pod.status.phase)
            return resource.Resource(
                type=self._SUPPORT_RESOURCE_TYPE,
                id=uuid.uuid5(uuid.NAMESPACE_OID, "%s:%s" % (
                    self.namespace, pod.metadata.name)),
                name="%s:%s" % (pod_namespace, pod.metadata.name),
                extra_info={'namespace': pod_namespace})
```

### 5. get_dependent_resources: 
 This function will be called for every parent resource type. 
```python
    def get_dependent_resources(self, context, parent_resource):
        self.list_resources(context)
```

## The pod protection plugin in Karbor

Protection plugins are one of the core components of Karbor's protection service. During protect, restore, and delete operations, Karbor is activating the protection plugins for the relevant resources in a certain order. Each protection plugin can handle one or more protectables (resource types) and specifices the actual implementation for them.

### Operation
In order to specify the detailed flow of each operation, a *Protection Plugin*
needs to return an Operation instance implementing 'hooks'. 

on_prepare_begin: invoked before any hook of this resource and dependent resources has begun.

on_prepare_finish: invoked after any prepare hooks of dependent resources are complete.

on_main: invoked after the resource *prepare hooks* are complete.

on_complete: invoked once the resource's main hook is completed, and the dependent resources' *complete hooks* are completed.


### 1. Protect Operation: 
This function will create a checkpoint from an existing resource.
```python
class ProtectOperation(protection_plugin.Operation):
    def on_main(self, checkpoint, resource, context, parameters, **kwargs):
        pod_id = resource.id
        pod_name = resource.name
        bank_section = checkpoint.get_resource_bank_section(pod_id)
        k8s_client = ClientFactory.create_client("k8s", context)
        resource_definition = {"resource_id": pod_id}

        LOG.info("Creating pod backup, id: %(pod_id)s) name: "
                 "%(pod_name)s.", {"pod_id": pod_id, "pod_name": pod_name})
        try:
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_PROTECTING)

            # get metadata about pod
            pod_namespace, k8s_pod_name = pod_name.split(":")
            pod = k8s_client.read_namespaced_pod(
                k8s_pod_name, pod_namespace)
            resource_definition["resource_name"] = pod_name
            resource_definition["namespace"] = pod_namespace

            mounted_volumes_list = self._get_mounted_volumes(
                k8s_client, pod, pod_namespace)
            containers_list = self._get_containers(pod)

            # save all pod's metadata
            pod_metadata = {
                'apiVersion': pod.api_version,
                'kind': pod.kind,
                'metadata': {
                    'labels': pod.metadata.labels,
                    'name': pod.metadata.name,
                    'namespace': pod.metadata.namespace,
                },
                'spec': {
                    'containers': containers_list,
                    'volumes': mounted_volumes_list,
                    'restartPolicy': pod.spec.restart_policy
                }
            }
            resource_definition["pod_metadata"] = pod_metadata
            LOG.debug("Creating pod backup, pod_metadata: %s.",
                      pod_metadata)
            bank_section.update_object("metadata", resource_definition)
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_AVAILABLE)
            LOG.info("Finish backup pod, pod_id: %s.", pod_id)
        except Exception as err:
            LOG.exception("Create pod backup failed, pod_id: %s.", pod_id)
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_ERROR)
            raise exception.CreateResourceFailed(
                name="Pod Backup",
                reason=err,
                resource_id=pod_id,
                resource_type=constants.POD_RESOURCE_TYPE)
```

### 2. Restore Operation: 
This function will create a resource from an existing checkpoint.
```python
class DeleteOperation(protection_plugin.Operation):
    def on_main(self, checkpoint, resource, context, parameters, **kwargs):
        resource_id = resource.id
        bank_section = checkpoint.get_resource_bank_section(resource_id)

        LOG.info("Deleting pod backup, pod_id: %s.", resource_id)

        try:
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_DELETING)
            objects = bank_section.list_objects()
            for obj in objects:
                if obj == "status":
                    continue
                bank_section.delete_object(obj)
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_DELETED)
            LOG.info("Finish delete pod, pod_id: %s.", resource_id)
        except Exception as err:
            LOG.error("Delete backup failed, pod_id: %s.", resource_id)
            bank_section.update_object("status",
                                       constants.RESOURCE_STATUS_ERROR)
            raise exception.DeleteResourceFailed(
                name="Pod Backup",
                reason=err,
                resource_id=resource_id,
                resource_type=constants.POD_RESOURCE_TYPE)
```

### 3. Delete Operation: 
This function will delete the resource from a checkpoint.
```python
class RestoreOperation(protection_plugin.Operation):
    def __init__(self, poll_interval):
        super(RestoreOperation, self).__init__()
        self._interval = poll_interval

    def on_complete(self, checkpoint, resource, context, parameters, **kwargs):
        original_pod_id = resource.id
        LOG.info("Restoring pod backup, pod_id: %s.", original_pod_id)

        update_method = None
        try:
            resource_definition = checkpoint.get_resource_bank_section(
                original_pod_id).get_object("metadata")

            LOG.debug("Restoring pod backup, metadata: %s.",
                      resource_definition)

            k8s_client = ClientFactory.create_client("k8s", context)
            new_resources = kwargs.get("new_resources")

            # restore pod
            new_pod_name = self._restore_pod_instance(
                k8s_client, new_resources, original_pod_id,
                parameters.get(
                    "restore_name",
                    "karbor-restored-pod-%s" % uuidutils.generate_uuid()),
                resource_definition)

            update_method = partial(utils.update_resource_restore_result,
                                    kwargs.get('restore'), resource.type,
                                    new_pod_name)
            update_method(constants.RESOURCE_STATUS_RESTORING)
            pod_namespace = resource_definition["namespace"]
            self._wait_pod_to_running(k8s_client, new_pod_name,
                                      pod_namespace)

            new_resources[original_pod_id] = new_pod_name
            update_method(constants.RESOURCE_STATUS_AVAILABLE)
            LOG.info("Finish restore pod, pod_id: %s.",
                     original_pod_id)

        except Exception as e:
            if update_method:
                update_method(constants.RESOURCE_STATUS_ERROR, str(e))
            LOG.exception("Restore pod backup failed, pod_id: %s.",
                          original_pod_id)
            raise exception.RestoreResourceFailed(
                name="Pod Backup",
                reason=e,
                resource_id=original_pod_id,
                resource_type=constants.POD_RESOURCE_TYPE
            )
```

### 4. get_supported_resources_types: 
This method should return a list of resource types this plugin handles. The plugin's methods will be called for each resource of these types. For example: `OS::Nova::Instance`,`OS::Cinder::Volume`.
```python
    _SUPPORT_RESOURCE_TYPES = [constants.POD_RESOURCE_TYPE]
    @classmethod
    def get_supported_resources_types(cls):
        return cls._SUPPORT_RESOURCE_TYPES
```

### 5. get_options_schema: 
This method should returns a schema of options and parameters for a protect operation.
```python
    @classmethod
    def get_options_schema(cls, resource_type):
        return pod_plugin_schemas.OPTIONS_SCHEMA
```

## Protect the pod with mounted volumes in Karbor

### 1 Configuration

Add the configuration about the kubernetes authentication to Karbor.conf

```bash
vi /etc/karbor/karbor.conf
[k8s_client]
k8s_host="https://192.168.98.35:6443/"
k8s_ssl_ca_cert=/etc/karbor/providers.d/server-ca.crt
k8s_cert_file=/etc/karbor/providers.d/client-admin.crt
k8s_key_file=/etc/karbor/providers.d/client-admin.key
``` 

Add the alias of the pod protection plugin to provider.conf
```bash
vi /etc/karbor/providers.d/openstack-infra.conf
[provider]
name = OS Infra Provider
description = This provider uses OpenStack's own services (swift, cinder) as storage
id = cf56bd3e-97a7-4078-b6d5-f36246333fd9

plugin=karbor-volume-protection-plugin
plugin=karbor-image-protection-plugin
plugin=karbor-server-protection-plugin
plugin=karbor-share-protection-plugin
plugin=karbor-network-protection-plugin
plugin=karbor-database-protection-plugin
plugin=karbor-pod-protection-plugin
bank=karbor-swift-bank-plugin

enabled=True

[swift_client]
swift_auth_url=http://127.0.0.1/identity
swift_user=demo
swift_key=password
swift_tenant_name=demo

[swift_bank_plugin]
lease_expire_window=120
lease_renew_window=100
lease_validity_window=100

```

### 1 Get the pod of the kubernetes from Karbor
```bash
root@ubuntu:/opt# karbor protectable-list-instances OS::Kubernetes::Pod
+--------------------------------------+---------------------+----------------------+---------------------+----------------------------+
| Id                                   | Type                | Name                 | Dependent resources | Extra info                 |
+--------------------------------------+---------------------+----------------------+---------------------+----------------------------+
| c88b92a8-e8b4-504c-bad4-343d92061871 | OS::Kubernetes::Pod | default:busybox-test | []                  | {u'namespace': u'default'} |
+--------------------------------------+---------------------+----------------------+---------------------+----------------------------+
root@ubuntu:/opt# karbor protectable-show-instance OS::Kubernetes::Pod c88b92a8-e8b4-504c-bad4-343d92061871 --parameters name=busybox-test
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| dependent_resources | []                                   |
| extra_info          | {u'namespace': u'default'}           |
| id                  | c88b92a8-e8b4-504c-bad4-343d92061871 |
| name                | default:busybox-test                 |
| type                | OS::Kubernetes::Pod                  |
+---------------------+--------------------------------------+

root@ubuntu:/opt# karbor protectable-show-instance OS::Kubernetes::Pod c88b92a8-e8b4-504c-bad4-343d92061871 --parameters name=busybox-test
+---------------------+-----------------------------------------------------------------------------+
| Property            | Value                                                                       |
+---------------------+-----------------------------------------------------------------------------+
| dependent_resources | [                                                                           |
|                     |   {                                                                         |
|                     |     "extra_info": {                                                         |
|                     |       "availability_zone": "nova"                                           |
|                     |     },                                                                      |
|                     |     "id": "7daedb1d-fc99-4a35-ab1b-b64971271d17",                           |
|                     |     "name": "kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097",  |
|                     |     "type": "OS::Cinder::Volume"                                            |
|                     |   }                                                                         |
|                     | ]                                                                           |
| extra_info          | {u'namespace': u'default'}                                                  |
| id                  | c88b92a8-e8b4-504c-bad4-343d92061871                                        |
| name                | default:busybox-test                                                        |
| type                | OS::Kubernetes::Pod                                                         |
+---------------------+-----------------------------------------------------------------------------+

```

### 2 Create a plan with a pod
```bash
root@ubuntu:/usr/local/lib/python2.7/dist-packages/karborclient# karbor plan-create 'OS pod protection plan77.' 'cf56bd3e-97a7-4078-b6d5-f36246333fd9' 'c88b92a8-e8b4-504c-bad4-343d92061871'='OS::Kubernetes::Pod'='default:busybox-test'
+-------------+----------------------------------------------------+
| Property    | Value                                              |
+-------------+----------------------------------------------------+
| description | None                                               |
| id          | 0e94d423-e9fd-4197-b706-e423e54a1274               |
| name        | OS pod protection plan77.                          |
| parameters  | {}                                                 |
| provider_id | cf56bd3e-97a7-4078-b6d5-f36246333fd9               |
| resources   | [                                                  |
|             |   {                                                |
|             |     "id": "c88b92a8-e8b4-504c-bad4-343d92061871",  |
|             |     "name": "default:busybox-test",                |
|             |     "type": "OS::Kubernetes::Pod"                  |
|             |   }                                                |
|             | ]                                                  |
| status      | suspended                                          |
+-------------+----------------------------------------------------+

root@ubuntu:/opt# cinder list
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status | Name                                                        | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| 7daedb1d-fc99-4a35-ab1b-b64971271d17 | in-use | kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097 | 1    | lvmdriver-1 | false    | 2ca1e154-6add-4e3e-90e4-e6e9907aae72 |
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
root@ubuntu:/opt# cinder backup-list
+----+-----------+--------+------+------+--------------+-----------+
| ID | Volume ID | Status | Name | Size | Object Count | Container |
+----+-----------+--------+------+------+--------------+-----------+
+----+-----------+--------+------+------+--------------+-----------+
root@ubuntu:/opt# karbor plan-list
+--------------------------------------+---------------------------+-------------+--------------------------------------+-----------+
| Id                                   | Name                      | Description | Provider id                          | Status    |
+--------------------------------------+---------------------------+-------------+--------------------------------------+-----------+
| 0e94d423-e9fd-4197-b706-e423e54a1274 | OS pod protection plan77. | -           | cf56bd3e-97a7-4078-b6d5-f36246333fd9 | suspended |
+--------------------------------------+---------------------------+-------------+--------------------------------------+-----------+
```
### 3 Create a checkpoint from the plan
```bash
root@ubuntu:/opt# karbor checkpoint-create cf56bd3e-97a7-4078-b6d5-f36246333fd9 0e94d423-e9fd-4197-b706-e423e54a1274
+-----------------+------------------------------------------------------+
| Property        | Value                                                |
+-----------------+------------------------------------------------------+
| created_at      | None                                                 |
| extra_info      | {"created_by": "manual"}                             |
| id              | 289ad047-8129-4e69-bf24-971ea95d7b5e                 |
| project_id      | 5004e1415225483681f672cb8a8b01a2                     |
| protection_plan | {                                                    |
|                 |   "id": "0e94d423-e9fd-4197-b706-e423e54a1274",      |
|                 |   "name": "OS pod protection plan77.",               |
|                 |   "resources": [                                     |
|                 |     {                                                |
|                 |       "extra_info": null,                            |
|                 |       "id": "c88b92a8-e8b4-504c-bad4-343d92061871",  |
|                 |       "name": "default:busybox-test",                |
|                 |       "type": "OS::Kubernetes::Pod"                  |
|                 |     }                                                |
|                 |   ]                                                  |
|                 | }                                                    |
| resource_graph  | None                                                 |
| status          | protecting                                           |
+-----------------+------------------------------------------------------+
root@ubuntu:/opt# cinder list
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status | Name                                                        | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| 7daedb1d-fc99-4a35-ab1b-b64971271d17 | in-use | kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097 | 1    | lvmdriver-1 | false    | 2ca1e154-6add-4e3e-90e4-e6e9907aae72 |
+--------------------------------------+--------+-------------------------------------------------------------+------+-------------+----------+--------------------------------------+
root@ubuntu:/opt# cinder backup-list
+--------------------------------------+--------------------------------------+-----------+------+------+--------------+---------------+
| ID                                   | Volume ID                            | Status    | Name | Size | Object Count | Container     |
+--------------------------------------+--------------------------------------+-----------+------+------+--------------+---------------+
| 05f01aec-b103-4a63-a71a-e0dcf5ad78f4 | 7daedb1d-fc99-4a35-ab1b-b64971271d17 | available | -    | 1    | 22           | volumebackups |
+--------------------------------------+--------------------------------------+-----------+------+------+--------------+---------------+
root@ubuntu:/opt# karbor checkpoint-show cf56bd3e-97a7-4078-b6d5-f36246333fd9 289ad047-8129-4e69-bf24-971ea95d7b5e
+-----------------+-----------------------------------------------------------------------+
| Property        | Value                                                                 |
+-----------------+-----------------------------------------------------------------------+
| created_at      | 2017-09-07                                                            |
| extra_info      | {"created_by": "manual"}                                              |
| id              | 289ad047-8129-4e69-bf24-971ea95d7b5e                                  |
| project_id      | 5004e1415225483681f672cb8a8b01a2                                      |
| protection_plan | {                                                                     |
|                 |   "id": "0e94d423-e9fd-4197-b706-e423e54a1274",                       |
|                 |   "name": "OS pod protection plan77.",                                |
|                 |   "provider_id": "cf56bd3e-97a7-4078-b6d5-f36246333fd9",              |
|                 |   "resources": [                                                      |
|                 |     {                                                                 |
|                 |       "extra_info": null,                                             |
|                 |       "id": "c88b92a8-e8b4-504c-bad4-343d92061871",                   |
|                 |       "name": "default:busybox-test",                                 |
|                 |       "type": "OS::Kubernetes::Pod"                                   |
|                 |     }                                                                 |
|                 |   ]                                                                   |
|                 | }                                                                     |
| resource_graph  | [                                                                     |
|                 |   {                                                                   |
|                 |     "0x0": [                                                          |
|                 |       "OS::Cinder::Volume",                                           |
|                 |       "7daedb1d-fc99-4a35-ab1b-b64971271d17",                         |
|                 |       "kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097",  |
|                 |       {                                                               |
|                 |         "availability_zone": "nova"                                   |
|                 |       }                                                               |
|                 |     ],                                                                |
|                 |     "0x1": [                                                          |
|                 |       "OS::Kubernetes::Pod",                                          |
|                 |       "c88b92a8-e8b4-504c-bad4-343d92061871",                         |
|                 |       "default:busybox-test",                                         |
|                 |       null                                                            |
|                 |     ]                                                                 |
|                 |   },                                                                  |
|                 |   [                                                                   |
|                 |     [                                                                 |
|                 |       "0x1",                                                          |
|                 |       [                                                               |
|                 |         "0x0"                                                         |
|                 |       ]                                                               |
|                 |     ]                                                                 |
|                 |   ]                                                                   |
|                 | ]                                                                     |
| status          | available                                                             |
+-----------------+-----------------------------------------------------------------------+
```

### 4 Restore a checkpoint
```bash
root@ubuntu:/opt# karbor restore-create cf56bd3e-97a7-4078-b6d5-f36246333fd9 289ad047-8129-4e69-bf24-971ea95d7b5e
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checkpoint_id    | 289ad047-8129-4e69-bf24-971ea95d7b5e |
| id               | fb186d7a-7f07-45bd-8c87-cf4e8c249f55 |
| parameters       | {}                                   |
| project_id       | 5004e1415225483681f672cb8a8b01a2     |
| provider_id      | cf56bd3e-97a7-4078-b6d5-f36246333fd9 |
| resources_reason | {}                                   |
| resources_status | {}                                   |
| restore_target   | None                                 |
| status           | in_progress                          |
+------------------+--------------------------------------+
root@ubuntu:/opt# 
root@ubuntu:/opt# karbor restore-list
+--------------------------------------+----------------------------------+--------------------------------------+--------------------------------------+----------------+------------+---------+
| Id                                   | Project id                       | Provider id                          | Checkpoint id                        | Restore target | Parameters | Status  |
+--------------------------------------+----------------------------------+--------------------------------------+--------------------------------------+----------------+------------+---------+
| fb186d7a-7f07-45bd-8c87-cf4e8c249f55 | 5004e1415225483681f672cb8a8b01a2 | cf56bd3e-97a7-4078-b6d5-f36246333fd9 | 289ad047-8129-4e69-bf24-971ea95d7b5e | -              | {}         | success |
+--------------------------------------+----------------------------------+--------------------------------------+--------------------------------------+----------------+------------+---------+

root@ubuntu:/opt# karbor restore-show fb186d7a-7f07-45bd-8c87-cf4e8c249f55
+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Property         | Value                                                                                                                                                                     |
+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checkpoint_id    | 289ad047-8129-4e69-bf24-971ea95d7b5e                                                                                                                                      |
| id               | fb186d7a-7f07-45bd-8c87-cf4e8c249f55                                                                                                                                      |
| parameters       | {}                                                                                                                                                                        |
| project_id       | 5004e1415225483681f672cb8a8b01a2                                                                                                                                          |
| provider_id      | cf56bd3e-97a7-4078-b6d5-f36246333fd9                                                                                                                                      |
| resources_reason | {u'OS::Cinder::Volume#892c4146-3da8-4362-8f99-b63eb06e4226': u'', u'OS::Kubernetes::Pod#karbor-restored-pod-09d4e588-6ca5-42de-826d-3ee43e5ff364': u''}                   |
| resources_status | {u'OS::Cinder::Volume#892c4146-3da8-4362-8f99-b63eb06e4226': u'available', u'OS::Kubernetes::Pod#karbor-restored-pod-09d4e588-6ca5-42de-826d-3ee43e5ff364': u'available'} |
| restore_target   | None                                                                                                                                                                      |
| status           | success                                                                                                                                                                   |
+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
root@ubuntu:/opt# 
root@ubuntu:/opt# cinder list
+--------------------------------------+--------+---------------------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status | Name                                                                      | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+--------+---------------------------------------------------------------------------+------+-------------+----------+--------------------------------------+
| 7daedb1d-fc99-4a35-ab1b-b64971271d17 | in-use | kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097               | 1    | lvmdriver-1 | false    | 2ca1e154-6add-4e3e-90e4-e6e9907aae72 |
| 892c4146-3da8-4362-8f99-b63eb06e4226 | in-use | 289ad047-8129-4e69-bf24-971ea95d7b5e@7daedb1d-fc99-4a35-ab1b-b64971271d17 | 1    | lvmdriver-1 | false    | 2ca1e154-6add-4e3e-90e4-e6e9907aae72 |
+--------------------------------------+--------+---------------------------------------------------------------------------+------+-------------+----------+--------------------------------------+
root@ubuntu:/opt# 
root@ubuntu:/opt# karbor protectable-list-instances OS::Kubernetes::Pod
+--------------------------------------+---------------------+------------------------------------------------------------------+-------------------------------------------------------------------------------------------+----------------------------+
| Id                                   | Type                | Name                                                             | Dependent resources                                                                       | Extra info                 |
+--------------------------------------+---------------------+------------------------------------------------------------------+-------------------------------------------------------------------------------------------+----------------------------+
| bcad8262-099d-5ef7-9a27-3310842e2d33 | OS::Kubernetes::Pod | default:karbor-restored-pod-09d4e588-6ca5-42de-826d-3ee43e5ff364 | [                                                                                         | {u'namespace': u'default'} |
|                                      |                     |                                                                  |   {                                                                                       |                            |
|                                      |                     |                                                                  |     "extra_info": {                                                                       |                            |
|                                      |                     |                                                                  |       "availability_zone": "nova"                                                         |                            |
|                                      |                     |                                                                  |     },                                                                                    |                            |
|                                      |                     |                                                                  |     "id": "892c4146-3da8-4362-8f99-b63eb06e4226",                                         |                            |
|                                      |                     |                                                                  |     "name": "289ad047-8129-4e69-bf24-971ea95d7b5e@7daedb1d-fc99-4a35-ab1b-b64971271d17",  |                            |
|                                      |                     |                                                                  |     "type": "OS::Cinder::Volume"                                                          |                            |
|                                      |                     |                                                                  |   }                                                                                       |                            |
|                                      |                     |                                                                  | ]                                                                                         |                            |
| c88b92a8-e8b4-504c-bad4-343d92061871 | OS::Kubernetes::Pod | default:busybox-test                                             | [                                                                                         | {u'namespace': u'default'} |
|                                      |                     |                                                                  |   {                                                                                       |                            |
|                                      |                     |                                                                  |     "extra_info": {                                                                       |                            |
|                                      |                     |                                                                  |       "availability_zone": "nova"                                                         |                            |
|                                      |                     |                                                                  |     },                                                                                    |                            |
|                                      |                     |                                                                  |     "id": "7daedb1d-fc99-4a35-ab1b-b64971271d17",                                         |                            |
|                                      |                     |                                                                  |     "name": "kubernetes-dynamic-pvc-fec036b7-9123-11e7-a930-fa163e18e097",                |                            |
|                                      |                     |                                                                  |     "type": "OS::Cinder::Volume"                                                          |                            |
|                                      |                     |                                                                  |   }                                                                                       |                            |
|                                      |                     |                                                                  | ]                                                                                         |                            |
+--------------------------------------+---------------------+------------------------------------------------------------------+-------------------------------------------------------------------------------------------+----------------------------+
```