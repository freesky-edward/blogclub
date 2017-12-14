# What's the details of share network in manila

# What's the DHSS in manila

OpenStack Manila provides secure filesystem storage over the network.
In multi-tenant environments, it is expected that file shares are
isolated between different tenants in the data path. Achieving strong
guarantees for data path isolation requires back end support, and cloud
administrators can configure their deployments to provide such isolation.
We create an option named "driver_handles_share_servers" (DHSS) which
defines the mode that back end drivers support.

# Create share with DHSS by CLI

We could reference to official way to create a share with DHSS by
[CLI](https://docs.openstack.org/mitaka/install-guide-ubuntu/launch-instance-manila-dhss-true-option2.html)


# Enviroment
My openstack enviroment is installed by devstack and use generic driver
with private network.


Config in generic

```
[generic1]
admin_subnet_id = 2c745913-7766-45b3-abfc-33a527e5185b
admin_network_id = dd07dbd2-cb02-4ef3-a8c0-5b3e75013c92
driver_handles_share_servers = True
service_instance_user = manila
service_password = manila
service_image_name = manila-service-image-master
path_to_private_key = /opt/stack/.ssh/id_rsa
path_to_public_key = /opt/stack/.ssh/id_rsa.pub
share_backend_name = GENERIC1
share_driver = manila.share.drivers.generic.GenericShareDriver
```

# Details about network

In creating the share workflow. We are going to create an instance as
the infrastructure for share server. The share server used for isolated the
network of the share. After the share verver has been created, we are going
to mount a cinder volume to that instance, the cinder volume which is going
to be created by the manila, and export the volume. After that the user can
directly use the path of the volume mount to any other instances which has the
ability to access the share.


1. Get service image

We are going to get the image object by glance image list api. The
image name is defined in generic driver option: ``service_image_name``.

```python
    def _get_service_image(self, context):
        """Returns ID of service image for service vm creating."""
        service_image_name = self.get_config_option("service_image_name")
        images = [image.id for image in self.compute_api.image_list(context)
                  if image.name == service_image_name]
```

2. Import the key pair by nova API in order to access your instance if the instance
doesn't have the password at initialization.
The key pair path is configed in ``path_to_public_key`` and
``path_to_private_key`` option by back end node.
The key pair info name is configed in ``manila_service_keypair_name`` option
in default node.

```python
    def _get_key(self, context):
        """Get ssh key.

        :param context: defines context, that should be used
        :returns: tuple with keypair name and path to private key.
        """
        if not (self.path_to_public_key and self.path_to_private_key):
            return (None, None)
        path_to_public_key = os.path.expanduser(self.path_to_public_key)
        path_to_private_key = os.path.expanduser(self.path_to_private_key)
        if (not os.path.exists(path_to_public_key) or
                not os.path.exists(path_to_private_key)):
            return (None, None)
        keypair_name = self.get_config_option("manila_service_keypair_name")
        keypairs = [k for k in self.compute_api.keypair_list(context)
                    if k.name == keypair_name]
        if len(keypairs) > 1:
            raise exception.ServiceInstanceException(_('Ambiguous keypairs.'))

        public_key, __ = self._execute('cat', path_to_public_key)
        if not keypairs:
            keypair = self.compute_api.keypair_import(
                context, keypair_name, public_key)
        else:
            keypair = keypairs[0]
            if keypair.public_key != public_key:
                LOG.debug('Public key differs from existing keypair. '
                          'Creating new keypair.')
                self.compute_api.keypair_delete(context, keypair.id)
                keypair = self.compute_api.keypair_import(
                    context, keypair_name, public_key)

        return keypair.name, path_to_private_key
```

3. Setup network

We are going to create the service subnet under the neutron subnet within
share network object.

```python
   def setup_network(self, network_info):
        neutron_net_id = network_info['neutron_net_id']
        neutron_subnet_id = network_info['neutron_subnet_id']
        network_data = dict()
        subnet_name = ('service_subnet_for_handling_of_share_server_for_'
                       'tenant_subnet_%s' % neutron_subnet_id)

        if self.use_service_network:
            network_data['service_subnet'] = self._get_service_subnet(
                subnet_name)
            if not network_data['service_subnet']:
                network_data['service_subnet'] = (
                    self.neutron_api.subnet_create(
                        self.admin_project_id, self.service_network_id,
                        subnet_name, self._get_cidr_for_subnet()))
```

4. Get private router

Find a private router object by the subnet id and the subnet gateway ip.
The subnet which is existing in the share network.

There is an option named ``connect_share_server_to_tenant_network``, in order
to check whether it have to attach share server directly to share network.
If it is set to False, we are going to directly get router that already exist
in neutron by neutron network.
If it is set to True, we have to create a new port with neutron network.

```python
    def _get_private_router(self, neutron_net_id, neutron_subnet_id):
        """Returns router attached to private subnet gateway."""
        private_subnet = self.neutron_api.get_subnet(neutron_subnet_id)
        if not private_subnet['gateway_ip']:
            raise exception.ServiceInstanceException(
                _('Subnet must have gateway.'))
        private_network_ports = [p for p in self.neutron_api.list_ports(
                                 network_id=neutron_net_id)]
        for p in private_network_ports:
            fixed_ip = p['fixed_ips'][0]
            if (fixed_ip['subnet_id'] == private_subnet['id'] and
                    fixed_ip['ip_address'] == private_subnet['gateway_ip']):
                private_subnet_gateway_port = p
                break
        else:
            raise exception.ServiceInstanceException(
                _('Subnet gateway is not attached to the router.'))
        private_subnet_router = self.neutron_api.show_router(
            private_subnet_gateway_port['device_id'])
        return private_subnet_router
```

If we also have config options which include admin_network_id and admin_subnet_id.
We also have to create a new port for admin network.


5. Setup connectivity with service instances

We are going to use OVSInterface (manila.network.linux.interface.OVSInterfaceDriver)
to creates host port in service network and/or admin network, creating and setting
up required network devices.

If you have to change the interface driver to set up required network devices,
you could change the "interface_driver" option value.

6. Creates server instance

Create a share server with the network that you already created in the above
steps. It means you are going to create a nova VM as your share server.

```python
    service_instance = self.compute_api.server_create(
        context,
        name=instance_name,
        image=service_image_id,
        flavor=self.get_config_option("service_instance_flavor_id"),
        key_name=key_name,
        nics=network_data['nics'],
        availability_zone=CONF.storage_availability_zone,
        **create_kwargs)
```

7. Creates security group

Get or create security group for service instance. The security group is
configured in ``service_instance_security_group`` option. If it has no value.
It will create a new security group for service instance.

8. Add security group to server instance

Then we will add security group to server instance in order to let manila share
service can access to share instance.

```python
self.compute_api.add_security_group_to_server(
                    context, service_instance["id"], sg_id)
```

9. Creates share

Since we already have the share instance, we can create a share with share network.

We are going to create a cinder volume by cinder api, and attach cinder
volume to server instance, then mount cinder volume and create exports.

We can use the exports as share export location to the user. The user can easily
use the export location path mount to another VM, or share with other users.

