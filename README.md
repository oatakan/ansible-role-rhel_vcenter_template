# rhel_vcenter_template
This repo contains an Ansible role that builds a RHEL/CentOS VM template from an ISO file on VMware vcenter.
You can run this role as a part of CI/CD pipeline for building RHEL/CentOS templates on VMware vcenter from an ISO file.

> **_Note:_** This role is provided as an example only. Do not use this in production. You can fork/clone and add/remove steps for your environment based on your organization's security and operational requirements.

Requirements
------------

You need to have the following packages installed on your control machine:

- mkisofs
- genisoimage

Before you can use this role, you need to make sure you have RHEL/CentOS install media iso file uploaded to a datastore on your vcenter environment.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of roles that this role utilizes:

- oatakan.rhn
- oatakan.rhel_upgrade
- oatakan.rhel_template_build

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - name: create a vmware rhel template
      hosts: all
      gather_facts: False
      connection: local
      become: no
      vars:
        template_force: yes #overwrite existing template with the same name
        export_ovf: no # export the template to export domain upon creation
        local_account_password: ''
        local_administrator_password: ''
        distro_name: rhel8 # this needs to be one of the standard values see 'os_short_names' var
        template_vm_name: rhel81-x64-v1
        template_vm_guest_id: rhel8_64Guest
        template_vm_root_disk_size: 10
        template_vm_memory: 4096
        iso_file_name: '' # name of the iso file
        
        vcenter_datacenter: '' # name of the datacenter
        vcenter_cluster: '' # name of the cluster
        vcenter_resource_pool: '' # name of resource pool if applicable
        vcenter_folder: '' # name of the folder to keep the template
        vcenter_datastore: '' # name of the datastore where iso file resides
        
        template_vm_network_name: mgmt
        template_vm_ip_address: 192.168.10.95 # static ip is required
        template_vm_netmask: 255.255.255.0
        template_vm_gateway: 192.168.10.254
        template_vm_domain: example.com
        template_vm_dns_servers:
        - 8.8.4.4
        - 8.8.8.8
    
      roles:
        - oatakan.rhel_vcenter_template

License
-------

MIT

Author Information
------------------

Orcun Atakan
