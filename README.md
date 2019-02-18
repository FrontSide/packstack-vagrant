# Packstack challenge

## Solved

---

## Requirements
*  vagrant
*  virtualBox
*  ansible

## Tasks
1) Use the provided Vagrantfile to spin up a local OpenStack instance (NOTE:
you may need to modify the amount of RAM and vCPUs depending on the resources
available to you). Initially the deployment should fail. Add configuration to the Vagrantfile to fix this problem.

```
I re-enabled the defaultsync from the current foler to /vagrant inside the VM so that the answers.txt for packstack would become available.
This also required me to prepend the path to the answers.txt in the packstack command with "/vagrant"
```

2) At this point, your installation will still fail. This is because the
OpenStack Compute Service, Nova, has not been installed. Update the configuration to fix this. The full installation can take ~15 minutes, though this will depend on the resources you're able to allocate to the VM.

```
In answers,txt I changed the value of CONFIG_INSTALL_NOVA to "y" to install the compute service nova
```

3) The vagrant box runs a number of OpenStack services. We want to be able to
access these services from our local machine. Add configuration to the
Vagrantfile to allow this (You may find the `openstack endpoint list` command
helpful).

```
In order to be able to run the openstack endpoint list command, 
I had to first source keystonerc_admin as root user in its home directory

I am forwarding the keystone service on port 5000 onto my machine so I can
authenticate against openstack from outside the virtualbox vm.

Additionally to keystone we will also need to expose 
- nova on port 8774 so we can manage compute
- glance on 9292 to access images
- neutron on 9696 for networking
- cinder on 8776 for volumes
```

4) Create an Ansible project which contains the following:
  * Custom role(s) that can be used to:
    * Spin up VMs, create networks, subnets and flavors

```
I decided to use the openstack ansible cloud modules which allow for easy creation  of OS resources. 
In order to use these modules it is necessary to have shade (openstack client lib) 
installed (even though the ansible module documentation says openstacksdk is required)

pip install shade 

I added separate roles for each of the above use cases.
One of the pitfalls I encountered was that I had to add user_domain_name
and project_domain_name parameters to the auth section in the ansible task
in order to be able to connect via keystone.
```

  * An Ansible playbook to orchestrate the creation of virtual machines using the custom roles.

```
Playbooks for each use case now allow me to perform tasks such as
create a VM with: ansible-playbook create-vm.yml

After running all playbooks separately I was able to see my new resources
with openstack <resource> list commands:

[root@packstack ~(keystone_admin)]# openstack server list                                         
+--------------------------------------+-------+---------+--------------------+--------+---------+
| ID                                   | Name  | Status  | Networks           | Image  | Flavor  |
+--------------------------------------+-------+---------+--------------------+--------+---------+
| 4aa51fbc-fbf4-4a54-b7e6-f37a60456b71 | vm1   | SHUTOFF | public=172.24.4.21 | cirros | m1.tiny |
| 1f086d98-3e91-4616-8cf1-0561c9ee4a31 | first | SHUTOFF | public=172.24.4.4  | cirros | m1.tiny |
+--------------------------------------+-------+---------+--------------------+--------+---------+

For some reason the VMs went into status SHUTOFF

[root@packstack ~(keystone_admin)]# openstack network list                                     
+--------------------------------------+---------------+--------------------------------------+
| ID                                   | Name          | Subnets                              |
+--------------------------------------+---------------+--------------------------------------+
| b4344d80-e8bf-4033-a594-291966ceeda0 | public_custom | 2cdfb840-b56c-4126-bee2-f5d69601a87a |
| c9fa79a7-decb-4968-b711-2d5466bfbbc8 | private       | 8779d89c-61ab-4642-8373-105fcf473ba6 |
| dc4d28f0-37b9-4683-9ffe-45cfa91a43f7 | public        | 771031f2-af56-4cb0-a093-7babda86b1dd |
+--------------------------------------+---------------+--------------------------------------+

[root@packstack ~(keystone_admin)]# openstack subnet list                                            
+--------------------------------------+----------------------+--------------------------------------+----------------+
| ID                                   | Name                 | Network                              | Subnet         |
+--------------------------------------+----------------------+--------------------------------------+----------------+                 
| 2cdfb840-b56c-4126-bee2-f5d69601a87a | public_custom_subnet | b4344d80-e8bf-4033-a594-291966ceeda0 | 192.168.0.0/24 |                 
| 771031f2-af56-4cb0-a093-7babda86b1dd | public_subnet        | dc4d28f0-37b9-4683-9ffe-45cfa91a43f7 | 172.24.4.0/24  |                 
| 8779d89c-61ab-4642-8373-105fcf473ba6 | private_subnet       | c9fa79a7-decb-4968-b711-2d5466bfbbc8 | 10.0.0.0/24    |
+--------------------------------------+----------------------+--------------------------------------+----------------+                

[root@packstack ~(keystone_admin)]# openstack flavor list                                            
+--------------------------------------+-------------+-------+------+-----------+-------+-----------+
| ID                                   | Name        |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+-------------+-------+------+-----------+-------+-----------+
| 1                                    | m1.tiny     |   512 |    1 |         0 |     1 | True      |
| 2                                    | m1.small    |  2048 |   20 |         0 |     1 | True      |
| 3                                    | m1.medium   |  4096 |   40 |         0 |     2 | True      |
| 4                                    | m1.large    |  8192 |   80 |         0 |     4 | True      |
| 5                                    | m1.xlarge   | 16384 |  160 |         0 |     8 | True      |
| a012945b-8f39-4372-9b4f-dadc470eb643 | tiny_custom |  1024 |    5 |         5 |     1 | True      |
+--------------------------------------+-------------+-------+------+-----------+-------+-----------+
```

  * Use YAML to configure the number of Virtual Machines, and their specs.

``` 
I decided to use group_vars/all.yml as the shared YAML configuration for the playbook and roles.
In this file I defined the variables vm_count and vm_flavor to specify the number of vms to be crated and 
the flavor to be selected.

(Originally I also put authentication for openstack into this file, however, I later
found that clouds.yml is a better place as it is also used for the inventory plugin)

The vm_count works in that the role to create a vm is executed n time through a "with_sequence" loop
in the create-vm playbook, whereas n == vm_count. Each VM is then named "vm<idx>" whereas idx is the 
loop index. So if vm_count==3 following VMs are created: vm1, vm2 and vm3.

[root@packstack ~(keystone_admin)]# openstack server list                                        
+--------------------------------------+------+--------+--------------------+--------+---------+ 
| ID                                   | Name | Status | Networks           | Image  | Flavor  | 
+--------------------------------------+------+--------+--------------------+--------+---------+ 
| 89a13787-3a6a-4b79-b95d-10dea09a8f45 | vm3  | ACTIVE | public=172.24.4.6  | cirros | m1.tiny | 
| 2c842d74-a333-4b33-a37f-c54c6a82bb83 | vm2  | ACTIVE | public=172.24.4.11 | cirros | m1.tiny | 
| c75663a9-c1c0-4f84-baa4-8f83ba4ab65e | vm1  | ACTIVE | public=172.24.4.3  | cirros | m1.tiny | 
+--------------------------------------+------+--------+--------------------+--------+---------+ 

Now they are ACTIVE as well

```

5) Create a dynamic Ansible inventory that can be used to:
    * Query Openstack and retrieve the Names, IP Addresses and other information from the Compute Service for Virtual Machines.
    
```
For the dynamic inventory I'm using the openstack inventory plugin 
(https://docs.ansible.com/ansible/latest/plugins/inventory/openstack.html)

- I added a cloud.yml to hold the credentials for keystone (also used by the previously created roles)
- A openstack.yml to hold the configuration for the openstack inventory plugin
- And a ansible.cfg to define openstack and openstack.yml as the default inventory
  so that I can omit the -i openstack.yml when running a playbook

I have one VM running on openstack, and it I now run ansible-inventory --graph it results in following output:

@all:
  |--@:
  |  |--vm1
  |--@_nova:
  |  |--vm1
  |--@default:
  |  |--vm1
  |--@default_:
  |  |--vm1
  |--@default__nova:
  |  |--vm1
  |--@flavor-m1.tiny:
  |  |--vm1
  |--@image-cirros:
  |  |--vm1
  |--@instance-88db5906-0af1-4b0d-867f-c7243d7b1c3d:
  |  |--vm1
  |--@nova:
  |  |--vm1
  |--@ungrouped:

If e.g. I now had a playbook with hosts: default, it would run against vm1.

Now, in order to retrieve more information about a VM. One can run (e.g. for vm1) 
$ ansible-inventory --host=vm1

{
    "ansible_host": "172.24.4.2",
    "ansible_ssh_host": "172.24.4.2",
    "openstack": {
        "OS-DCF:diskConfig": "MANUAL",
        "OS-EXT-AZ:availability_zone": "nova",
        "OS-EXT-SRV-ATTR:host": "packstack",
        "OS-EXT-SRV-ATTR:hypervisor_hostname": "packstack",
        "OS-EXT-SRV-ATTR:instance_name": "instance-00000001",
        "OS-EXT-STS:power_state": 1,
        "OS-EXT-STS:task_state": null,
        "OS-EXT-STS:vm_state": "active",
        "OS-SRV-USG:launched_at": "2019-02-17T15:26:07.000000",
        "OS-SRV-USG:terminated_at": null,
        "accessIPv4": "172.24.4.2",
        "accessIPv6": "",
        "addresses": {
            "public": [
                {
                    "OS-EXT-IPS-MAC:mac_addr": "fa:16:3e:4c:98:5c",
                    "OS-EXT-IPS:type": "fixed",
                    "addr": "172.24.4.2",
                    "version": 4
                }
            ]
        },
        "adminPass": null,
        "az": "nova",
        "cloud": "default",
        "config_drive": "",
        "created": "2019-02-17T15:25:41Z",
        "created_at": "2019-02-17T15:25:41Z",
        "disk_config": "MANUAL",
        "flavor": {
            "id": "1",
            "name": "m1.tiny"
        },
        
        ............... etc .......................

        "status": "ACTIVE",
        "task_state": null,
        "tenant_id": "37fdd4c4a6524caba1023b4d6f8a300b",
        "terminated_at": null,
        "updated": "2019-02-17T15:26:07Z",
        "user_id": "993093e9a36746fc94c1b81800308d73",
        "vm_state": "active",
        "volumes": []
    },
    "vm_count": 1,
    "vm_flavor": "m1.tiny"
}

```

* Group Virtual Machines (so they can be targeted by Ansible) based on some attributes, for example:
      * vm_state, power_state, flavor, host etc.

```

The in-development version of the openstack ansible inventory plugin 
allows us to modify the openstack.yml plugin configuration so that we can add 
keyed groups to it for creating groups based on the values of host attributes 
e.g. one per vm_state: active, stopped etc...

Unfortunately this version is not yet available in the latest released version of ansible, 
so we will need to use and inventory script.
The documentation for dynamic inventories already links to an inventory script, 
so I downloaded it and use it as my starting point: https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html#implicit-use-of-openstack-inventory-script

To use this new script as default inventory, rather than the plugin, 
I changed the ansible.cfg to use openstack_inventory.py.
And I removed the enabled_plugin line.

The commands to show the inventory graph and info about a host 
works just the same as before.

However, this now also means that openstack.yml doesn't get used anymore.
We could theoretically still use it by moving it into /etc/ansible, but we don't really need to.

For the custom key settings, I will add a dictionary to the openstack_inventory.py
file where a user can add attribute names of hosts, so that custom keys can be generated.

If I had more time I would suggest to improve the script so that 
it can read from a "keyed_groups" attribute in the openstack.yml file,
as the plugin is supposed to do.
Ideally, we would then just seamlessly swap over to the plugin once this 
feature is released with ansible.
I am also considering opening a PR for some improvements on the openstack inventory plugin
(mostly related to documentation).

The simpler (but less nice way) was to add the following:

    # To create custom server groupings by server attributes in the inventory, 
    # add an entry to the dict where the key is the prefix for the group and the 
    # value is the server variable key.
    #
    # e.g. to create new groupings based on the server's vm_state,
    # add an entry { "vm_state_": "vm_state" }, this will now create 
    # server groups based on the vm_state: "vm_state_active", "vm_state_stopped", etc.
    KEYED_GROUPS = {
        "vm_state_": "vm_state", 
        "power_state_": "power_state",
    }

and later in the code we parse this dict with:

    # Custom keyed_groups as per KEYED_GROUPS list 
    for keyed_group_prefix, host_var_key in KEYED_GROUPS.items():
        groups.append("%s%s" % (keyed_group_prefix, server_vars[host_var_key]))

If we now run ansible-inventory --graph, we get following output:

@all:
  |--@_nova:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@default:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@default_:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@default__nova:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@flavor-m1.tiny:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@image-cirros:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@instance-4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |--@instance-88db5906-0af1-4b0d-867f-c7243d7b1c3d:
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@nova:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@power_state_4:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@ungrouped:
  |--@vm1:
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d
  |--@vm2:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |--@vm_state_stopped:
  |  |--4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf
  |  |--88db5906-0af1-4b0d-867f-c7243d7b1c3d

As can be seen there is now a group for vm_state_stopped and for power_state_4.

We could now target our playbook to host: vm_state_stopped.

One more change that I made was to change the server listings in the inventory from 
id to name. This might not be suitable for all cases, but here it just makes it more readable.

@all:
  |--@_nova:
  |  |--vm1
  |  |--vm2
  |--@default:
  |  |--vm1
  |  |--vm2
  |--@default_:
  |  |--vm1
  |  |--vm2
  |--@default__nova:
  |  |--vm1
  |  |--vm2
  |--@flavor-m1.tiny:
  |  |--vm1
  |  |--vm2
  |--@image-cirros:
  |  |--vm1
  |  |--vm2
  |--@instance-4dc7abf4-dbe3-4829-888d-9bc1d1c0a1cf:
  |  |--vm2
  |--@instance-88db5906-0af1-4b0d-867f-c7243d7b1c3d:
  |  |--vm1
  |--@nova:
  |  |--vm1
  |  |--vm2
  |--@power_state_4:
  |  |--vm1
  |  |--vm2
  |--@ungrouped:
  |--@vm1:
  |  |--vm1
  |--@vm2:
  |  |--vm2
  |--@vm_state_stopped:
  |  |--vm1
  |  |--vm2


```

6) Upload all work to a public github repository and share the link in advance of your interview.

NOTE 1: Please note that your primary focus of this exercise should be on delivering the dynamic inventory requested in Step 5; ensure you have sufficient time to work on this.  
NOTE 2: It is not a requirement to get everything working; however we expect you to have made an effort for all sections and you should be able to explain your design choices. Some of the tasks are deliberately vague to allow you freedom to create your own solution. We are most interested in how you think and approach the problem.
