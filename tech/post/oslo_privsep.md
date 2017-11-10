# Oslo.privsep Introduce

This library helps applications perform actions which require more or less privileges than they were started with in a safe, easy to code and easy to use manner.

We used to use *rootwrap* to perform the privileged commands, but now Nova and Cinder are inclined to replace it with oslo.privsep for security and efficiency reasons. Let's compare them here and find out the pros and cons.

## Rootwrap
Rootwrap is introduced for the goal of allowing a service-specific unprivileged user to run a number of actions as the root user, it's widely used in the OpenStack services now. This process below shows
how a service can configure and use the rootwrap for their privileged
commands.

### Setup

1. Rootwrap always runs in a seperated process as a root user and only
exposes the command line for any service who wants to communicate with.
It's esay to find the rootwraps' entrance in the *setup.cfg*:
```
[entry_points]
    cinder-rootwrap = oslo_rootwrap.cmd:main
```

2. In the service itself, once we need to execute priveleged commands we have to ask rootwrap to perform and return. For instance, when Cinder
needs to setup a target in it's lvm driver, this is one of the commands
that needs privelege:
```
tgtadm --lld iscsi --op show --mode target
```
After appending the root helper's prefix the command will change into:
```
sudo cinder-rootwrap [rootwrap_config_file] tgtadm --lld iscsi --op....
```
The ``rootwrap_config_file`` is used for the rootwrap's generic config and filter invalid commands.
```
[DEFAULT]
#Comma-separated list of directories containing filter definition files
filters_path=/etc/cinder/rootwrap.d,/usr/share/cinder/rootwrap
#Comma-separated list of directories to search executables in
exec_dirs=/sbin,/usr/sbin,/bin,/usr/bin,/usr/local/bin,/usr/local/sbin,/usr/lpp/mmfs/b
#Enable logging to syslog. Default value is False
use_syslog=False
#Which syslog facility to use for syslog logging
syslog_log_facility=syslog
#Which messages to log
syslog_log_level=ERROR
```
### Filter

Rootwrap added command filter support in case of unexpected or unauthorized command are been used, it would deny the commands that don't match any filter. Some of the filters are:

1. **CommandFilter**: Basic filter that only checks the executable called
```
ploop: CommandFilter, ploop, root
[name]: CommandFilter, [command], [user]
```
2. **RegExpFilter**: Generic filter that checks the executable called, then uses a list of regular expressions to check all subsequent arguments
```
tunctl: RegExpFilter, /usr/sbin/tunctl, root, tunctl, -b, -t, .*
[name]: RegExpFilter, [command], [user], [command_regex, [option],...]
```

3. **PathFilter**: Generic filter that lets you check that paths provided as parameters fall under a given directory
```
chown: PathFilter, /bin/chown, root, nova, /var/lib/images
[name]: PathFilter, [command], [user], [path]
```

4. **EnvFilter**: Filter allowing extra environment variables to be set by the calling code
```
dnsmasq: EnvFilter, env, root, CONFIG_FILE=, NETWORK_ID=, dnsmasq
[name]: EnvFilter, env, [user], [env_config1, [env_config2]...] [command]
```
It also have ``ReadFileFilter``, ``IPFilter`` and ``IpNetnsExecFilter`` commands.
So back to our specific case, we need to add one more filter if we
want to execute the previous command.
````
[Filters]
# cinder/volume/iscsi.py: iscsi_helper '--op' ...
tgtadm: CommandFilter, tgtadm, root
````
When ``rootwrap`` is been executed, the program will parse the config file, load the filters and valid the input command at the very begining,and then create a sub-process to execute the command.

Based on the process above, we could find the disadvantage of rootwrap:

1. Efficiency: Generating command lines and parsing textual output from tools is slow.
2. Convenience: Service could have lots of privileged commands and it's hard to configure those filters correctly, because the filters have limited expressiveness.
3. Security: ``chown: CommandFilter, chown, root`` this command allows
the invoking user to run ``chown`` with any arguments as root. it could be dangerous if ``/etc/shadow`` is been passed.

Rootwrap team has tried to improve the efficiency by adding the
[daemon](https://specs.openstack.org/openstack/oslo-specs/specs/liberty/privsep.html) support in Liberty, but it still can not fix the Convenience and Security issues.

## Oslo.privsep

Privsep is designed based on the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege). It provides a fine-grained security control by Linux Capabilities rather than run the process as a superuser directly, and the client communicate with the privileged daemon through ``unix domain socket``, which is better than command line.

### Linux capabilities

The feature is added since kernel 2.2, linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled.  It's a per-thread attribute. 
Each thread has three capability sets containing zero or more of the capabilities.
1. **Permitted**: This is a limiting superset for the effective capabilities that the thread may assume.
2. **Inheritable**: This is a set of capabilities preserved across an ``execve``. Inheritable capabilities remain inheritable when executing any program, and inheritable capabilities are added to the permitted set when executing a program that has the corresponding bits set in the file inheritable set.
3. **Effective**: This is the set of capabilities used by the kernel to perform permission checks for the thread.

There are a various capabilities defined at the [linux man page](http://man7.org/linux/man-pages/man7/capabilities.7.html). For instance, ``CAP_SYS_ADMIN`` is used as the default capability for library [os_brick](https://github.com/openstack/os-brick)'s privsep daemon. It can provide a range of permissions including perfoming a range of system administration operations (quotactl, mount, umount,swapon, setdomainname).
We can diagnose process's capabilities by libcap (sudo aget-get install libcap-dev & getpcaps <PID>) library, this below is the nova-compute's capabilities:
```
Capabilities for `39036': = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,37+ep
```

### Using oslo.privsep

When using privsep in our service, we need to define the priveleged context and then use it as a decorator on our command execute code.

#### Define privsep context
```python 
default = priv_context.PrivContext(
    __name__,
    cfg_section='privsep_osbrick',
    pypath=__name__ + '.default',
    capabilities=[c.CAP_SYS_ADMIN],
).
```
By this code, we defined the configure section which will be used to initialize the deamon in the future and assign it within the required capabilities.  The configuration options are below:
```
[privsep_osbrick]
user = novapriv
group = novapriv
# This will overwrite the default capabilitie those are specified in the initialize code.
capabilities = CAP_SYS_ADMIN, CAP_NET_ADMIN 
```
When we execute the privelged command through privsep for the first time, privsep will start up the daemon process with the user, group and
capabilities.

### Use the privsep context
It's easy to make one of your commands to run with desired capabilties by using the privsep's decorator, see the ``entrypoint`` below:
```python
@privileged.default.entrypoint
def execute_root(*cmd, **kwargs):
    """NB: Raises processutils.ProcessExecutionError/OSError on failure."""
    return custom_execute(*cmd, shell=False, run_as_root=False, **kwargs)
```
And the whole magic comes from these two methods:
```python
    def entrypoint(self, func):
        """This is intended to be used as a decorator."""

        assert func.__module__.startswith(self.prefix), (
            '%r entrypoints must be below "%s"' % (self, self.prefix))

        # Right now, we only track a single context in
        # _ENTRYPOINT_ATTR.  This could easily be expanded into a set,
        # but that will increase the memory overhead.  Revisit if/when
        # someone has a need to associate the same entrypoint with
        # multiple contexts.
        assert getattr(func, _ENTRYPOINT_ATTR, None) is None, (
            '%r is already associated with another PrivContext' % func)

        f = functools.partial(self._wrap, func)
        setattr(f, _ENTRYPOINT_ATTR, self)
        return f
```
and
```python
    def _wrap(self, func, *args, **kwargs):
        if self.client_mode:
            name = '%s.%s' % (func.__module__, func.__name__)
            if self.channel is None:
                self.start()
            return self.channel.remote_call(name, args, kwargs)
        else:
            return func(*args, **kwargs)
```
With the transformation of the decorator any execute commands are been passed to the privsep daemon through ``linxu domain socket``. The daemon
will keep listening to the socket, executing the client's commands until the parent process exits.


