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
One of the pitfalls I encounterd was that I had to add user_domain_name
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
In this file I defined a nested os_auth variable which is used by all my openstack roles
to fetch authentication information.

Then I created variables vm_count and vm_flavor to specify the number of vms to be crated and 
the flavor to be selected.  

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
```

5) Create a dynamic Ansible inventory that can be used to:
    * Query Openstack and retrieve the Names, IP Addresses and other information from the Compute Service for Virtual Machines.
    * Group Virtual Machines (so they can be targeted by Ansible) based on some attributes, for example:
      * vm_state, power_state, flavor, host etc.


6) Upload all work to a public github repository and share the link in advance of your interview.

NOTE 1: Please note that your primary focus of this exercise should be on delivering the dynamic inventory requested in Step 5; ensure you have sufficient time to work on this.  
NOTE 2: It is not a requirement to get everything working; however we expect you to have made an effort for all sections and you should be able to explain your design choices. Some of the tasks are deliberately vague to allow you freedom to create your own solution. We are most interested in how you think and approach the problem.
