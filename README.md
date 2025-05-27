# rhel_vcenter_template
This repo contains an Ansible role that builds a RHEL/CentOS VM template from an ISO file on VMware vCenter.
You can run this role as a part of CI/CD pipeline for building RHEL/CentOS templates on VMware vCenter from an ISO file.

> **_Note:_** This role is provided as an example only. Do not use this in production. You can fork/clone and add/remove steps for your environment based on your organization's security and operational requirements.

## Supported Distributions
- Red Hat Enterprise Linux 7
- Red Hat Enterprise Linux 8  
- Red Hat Enterprise Linux 9
- **Red Hat Enterprise Linux 10** (New!)
- CentOS 7, 8, 9
- CentOS Stream 8, 9

## Requirements
------------

You need to have the following packages installed on your control machine:

- mkisofs
- genisoimage

Before you can use this role, you need to make sure you have RHEL/CentOS install media iso file uploaded to a datastore on your vCenter environment.

## Role Variables
--------------

### Core Variables
- `role_action`: Set to 'provision' or 'deprovision' (default: provision)
- `distro_name`: Target distribution - rhel7, rhel8, rhel9, rhel10 (default: rhel10)
- `template_vm_name`: Name for the created template (default: rhel10-x64-v1)
- `template_force`: Overwrite existing template with same name (default: false)
- `iso_file_name`: Name of the ISO file in the datastore

### RHEL 10 Specific Variables
- `template_vm_guest_id`: VM guest ID type (default: rhel9_64Guest for RHEL 10)
- `template_vm_secure_boot`: Enable secure boot for UEFI systems (default: false)
- `template_vm_controller`: SCSI controller type (default: lsilogicsas)

### Network Configuration
- `template_vm_network_name`: Network name in vCenter
- `template_vm_ip_address`: Static IP address (required)
- `template_vm_netmask`: Network subnet mask
- `template_vm_gateway`: Default gateway
- `template_vm_domain`: DNS domain name
- `template_vm_dns_servers`: List of DNS servers

### vCenter Configuration
- `vcenter_datacenter`: vCenter datacenter name
- `vcenter_cluster`: vCenter cluster name
- `vcenter_resource_pool`: Resource pool (optional)
- `vcenter_folder`: VM folder location
- `vcenter_datastore`: Datastore for template storage

### Template Customization
- `template_ks_name`: Custom kickstart file name (default: ks.cfg.j2)
- `enable_additional_roles`: Enable custom role execution (default: false)
- `additional_roles`: List of additional Ansible roles to execute
- `install_updates`: Install system updates during build (default: true)
- `build_template`: Run oatakan.rhel_template_build role (default: false)

## Dependencies
------------

Required roles for full functionality:
- oatakan.rhn
- oatakan.rhel_upgrade  
- oatakan.rhel_template_build

## Example Playbooks
----------------

### RHEL 10 Template Creation
```yaml
---
- name: create a vmware rhel 10 template
  hosts: all
  gather_facts: false
  connection: local
  become: false
  vars:
    template_force: yes # overwrite existing template with the same name
    export_ovf: no # export the template to export domain upon creation
    local_account_password: ''
    local_administrator_password: ''
    distro_name: rhel10
    template_vm_name: rhel10-x64-v1
    template_vm_guest_id: rhel9_64Guest
    template_vm_root_disk_size: 20
    template_vm_memory: 4096
    template_vm_cpu: 2
    template_vm_secure_boot: true
    iso_file_name: 'rhel-10.0-x86_64-dvd.iso'
    
    vcenter_datacenter: 'production'
    vcenter_cluster: 'cluster1'
    vcenter_resource_pool: 'templates'
    vcenter_folder: 'vm-templates'
    vcenter_datastore: 'shared-storage'
    
    template_vm_network_name: mgmt
    template_vm_ip_address: 192.168.10.95 # static ip is required
    template_vm_netmask: 255.255.255.0
    template_vm_gateway: 192.168.10.254
    template_vm_domain: example.com
    template_vm_dns_servers:
    - 8.8.4.4
    - 8.8.8.8

    install_updates: true
    build_template: true
    enable_additional_roles: true
    additional_roles:
      - mycompany.security_hardening
      - mycompany.monitoring_agent

  roles:
    - oatakan.rhel_vcenter_template
```

### RHEL 9 Template Creation (Backward Compatible)
```yaml
---
- name: create a vmware rhel 9 template
  hosts: all
  gather_facts: false
  connection: local
  become: false
  vars:
    template_force: yes
    distro_name: rhel9
    template_vm_name: rhel9-x64-v1
    template_vm_guest_id: rhel9_64Guest
    iso_file_name: 'rhel-9.3-x86_64-dvd.iso'
    # ... other variables

  roles:
    - oatakan.rhel_vcenter_template
```

### Template Deletion
```yaml
---
- name: delete a vmware rhel template
  hosts: all
  gather_facts: false
  connection: local
  become: false

  roles:
    - role: oatakan.rhel_vcenter_template
      role_action: deprovision
```

## Tags
-------

Available tags to run specific parts of the template build process:

- `preflight`: Run only the preflight checks
- `provision`: Run the full provisioning process
- `custom-roles`: Run only the defined custom roles, after initial build has completed
- `post-config`: Run the post configuration (unregister, convert to template)

### Example Usage with Tags
```bash
# Run only preflight checks
ansible-playbook site.yml --tags preflight

# Run only custom roles
ansible-playbook site.yml --tags custom-roles

# Skip custom roles
ansible-playbook site.yml --skip-tags custom-roles
```

## RHEL 10 Specific Features
--------------------------

### Enhanced Security
- Modern SSH configuration with strong ciphers and MACs
- Updated crypto policies (DEFAULT:SHA1)
- Enhanced systemd security features
- Improved SELinux configurations

### Package Management
- DNF5 as default package manager
- Optimized package selection for cloud/VM environments
- Enhanced dependency resolution

### System Optimizations
- Cloud-init integration for better cloud deployment
- Optimized XFS filesystem settings
- Modern systemd configurations
- Improved time synchronization with chronyd

### VMware Compatibility
- Uses rhel9_64Guest ID until VMware releases RHEL 10 specific guest tools
- Enhanced hardware compatibility settings
- Optimized for modern VMware vSphere environments

## Troubleshooting
-----------------

### Common Issues

1. **RHEL 10 ISO not found**
   - Ensure the ISO file is uploaded to the specified datastore
   - Verify the `iso_file_name` variable matches the actual file name

2. **Guest ID compatibility**
   - RHEL 10 currently uses `rhel9_64Guest` guest ID
   - Update to RHEL 10 specific guest ID when available from VMware

3. **Secure Boot issues**
   - Ensure vCenter supports secure boot for the target hardware version
   - Disable secure boot if experiencing boot issues

4. **Network configuration problems**
   - Verify static IP settings are correct for your environment
   - Ensure DNS servers are accessible from the target network

## License
-------

MIT

## Author Information
------------------

Orcun Atakan

## Contributing
--------------

1. Fork the repository
2. Create a feature branch
3. Make your changes following Ansible best practices
4. Test with ansible-lint
5. Submit a pull request

### Development Guidelines
- Follow existing code style and patterns
- Maintain backward compatibility
- Update documentation for new features
- Test with multiple RHEL versions when possible