# RHEL 7 Lab - Provision Red Hat Products on RHEL 7
## Introduction
This set of roles was written to support the bulk provisioning of RHEL 7 instances. Using this role it is trivial to provision 1 or 100 virtual machines. We support cleanup and retirement as well. The various roles currently provide integration with the following:

- phpIPAM
- IdM (FreeIPA)
- CloudForms (ManageIQ)
- RHV (oVirt)

### Product Roles
Building on-top of the core RHEL 7 Lab playbooks, there are several roles that can be used to install Red Hat products on-top of our newly minted instances. We currently support the following:

- Ansible Tower
- OpenShift (in-progress)
- CloudForms (in-progress)

## Requirements
Many of the modules we use require Ansible 2.4. We also require dnspython and python-ovirt-engine-sdk4 packages to be installed on the system that is executing the playbooks.

Naturally we are cloning templates as part of the provisioning process. For the roles to work properly (especially the IdM role), your template needs the following minimal configuration:

1. cloud-init and ipa-client packages installed
2. The cloud-init daemon enabled and running
3. ~~The firewalld  daemon should be enabled and running. Also, ssh should be blocked by default. Once the ipa-client-install command has successfully completed, we will re-enable ssh on the firewall using the cloud-cfg script and firewall-cmd. This is done to ensure the system is properly registered to IdM before the wait_for module is called.~~
4. TODO: Rework #3 using wait_for_connection (this would remove asinine firewall requirements)

## Global Variables

```
cfme_hostname: CloudForms Server with Web Services Role Enabled
cfme_username: Username for CloudForms (needs administrator privileges)
cfme_password: Password for CloudForms
rhv_hostname: RHV-M FQDN (NOT API URL)
rhv_username: Username in RHV-M admin@internal Format (needs administrator privileges)
rhv_password: Password for RHV-M
php_ipam_hostname: phpipam FQDN (NOT API URL)
php_ipam_username: Username for phpipam (needs administrator privileges)
php_ipam_password: Password for phpipam
php_ipam_api_user: API user for token
php_ipam_subnet_id: Subnet ID Used to Request Resources
ipa_username: Username for IdM (needs administrator privileges)
ipa_password: Password for IdM
ipa_hostname: IdM FQDN (NOT API URL)
host_dns_suffix: Used for Dynamic Inventory and build_ocp Roles
host_dns_id: Used for Dynamic Inventory and build_ocp Roles (unique ID to support multiple simultaneous environments)
ocp_call_byo_playbooks: false (Should always be false as BYO playbooks have not yet been incorporated)
```

## Integration Role Documentation

### phpipam

This role leverages the URI module to make a series of REST calls to phpipam. It also expects `number_vms` to be defined in the pre_tasks section of your initial play. Using the vaule defined in `number_vms`, Ansible will iterate through a sequence and make the appropriate number of REST calls until we have the required number of IPs. Lastly the role will build the `ip_addresses` list which can be used to form a syncronized list using `with_together`.

### dynamic_inventory

### idm

### rhv

### manageiq

### retire

## Product Role Documentation

### tower (Ansible Tower)

### ocp_inventory/ocp_call_byo (OpenShift v3.7)

## Example Playbook
```
---
- hosts: localhost
  name: Provision VM(s)
  connection: local
  gather_facts: false

  pre_tasks:
    - name: Convert host_csv to Array
      set_fact:
        hostnames: "{{ host_csv.split(',') }}"

    - debug: var=hostnames
      verbosity: 2

    - name: Get Host Array Length
      set_fact:
        number_vms: "{{ hostnames | length }}"

    - debug: var=number_vms
      verbosity: 2

    - name: Fail if number_vms > 5
      fail:
        msg: "Number of VMs too large: {{ number_vms }}"
      when: number_vms | int > 5

  roles:
    - role: phpipam
    - role: dynamic_inventory
    - role: idm
    - role: rhv
    - { role: manageiq, when: manageiq is defined }

# Example for use with dynamic_inventory (probably more suitable as another role)
- hosts: dynamic_inventory
  name: Configure VM(s)

  tasks:
    - name: Ensure SELinux is enabled/enforcing
      selinux:
        policy: targeted
        state: enforcing
```
