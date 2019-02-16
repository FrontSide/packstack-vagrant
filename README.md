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
2) At this point, your installation will still fail. This is because the
OpenStack Compute Service, Nova, has not been installed. Update the configuration to fix this. The full installation can take ~15 minutes, though this will depend on the resources you're able to allocate to the VM.
3) The vagrant box runs a number of OpenStack services. We want to be able to
access these services from our local machine. Add configuration to the
Vagrantfile to allow this (You may find the `openstack endpoint list` command
helpful).
4) Create an Ansible project which contains the following:
  * Custom role(s) that can be used to:
    * Spin up VMs, create networks, subnets and flavors
  * An Ansible playbook to orchestrate the creation of virtual machines using the custom roles.
  * Use YAML to configure the number of Virtual Machines, and their specs.
5) Create a dynamic Ansible inventory that can be used to:
    * Query Openstack and retrieve the Names, IP Addresses and other information from the Compute Service for Virtual Machines.
    * Group Virtual Machines (so they can be targeted by Ansible) based on some attributes, for example:
      * vm_state, power_state, flavor, host etc.
6) Upload all work to a public github repository and share the link in advance of your interview.

NOTE 1: Please note that your primary focus of this exercise should be on delivering the dynamic inventory requested in Step 5; ensure you have sufficient time to work on this.  
NOTE 2: It is not a requirement to get everything working; however we expect you to have made an effort for all sections and you should be able to explain your design choices. Some of the tasks are deliberately vague to allow you freedom to create your own solution. We are most interested in how you think and approach the problem.
