# Deploy RHEL to VMware Using Ansible and Image Builder

This Git repository contains a collection of Ansible playbooks designed to automate the deployment of Red Hat Enterprise Linux (RHEL) virtual machines on VMware infrastructure. Instead of using templates, images are generated at provisioning time utilizing the Hosted Image Builder service on console.redhat.com.

## Notable Features and Capabilities

* Use one image to deploy a variable number of VMs
* Integration with Red Hat Identity Management (IdM) during first boot
* If IdM is not available, support for initial user authentication via SSH keys
* Multiple NIC Support
* Multiple Disk Support

## Requirements

* Requires vCenter 6.7+, tested on vCenter 8.0u2
* Access to console.redhat.com
* Red Hat API Offline Token

## Defining VM and Image Properties

To being provisioning, we need to define an image and a VM (or set of VMs) to deploy. All of this information, including the image properties (e.g. the version of RHEL, any pre-installed packages, the partition layout) and VM scaffolding properties (like the CPU count, amount of RAM, NICs along with their settings, templated user data) can be stored in a single Ansible variables file. The following example shows the provisioning of multiple VMs that will ultimately support a Ceph cluster.

```yaml
# NOTE: Currently mock CPU/RAM requirements.
---
image_properties:
  # Initial user and SSH public key. This user will have full sudo access.
  cloud_user:
    public_key: |
      ssh-rsa AAAAB3Nza== cloud-user@lab.uc2.io
    user: cloud-user
  # RHEL distribution.
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-Distributions
  distribution: rhel-93
  # The name of the image.
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-ComposeRequest
  image_name: ceph-external-odf
  # image_requests should be left alone. Only `vsphere-ova` or `vsphere` are applicable, but 'vsphere' will generate a VMDK that does not support UEFI. As such, these playbooks only support `vsphere-ova`. 
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-ImageTypes
  image_requests:
    architecture: x86_64
    image_type: vsphere-ova
  # List of additional RPMs to install when the image is built
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-Customizations
  packages:
    - cloud-init
    - cloud-utils-growpart
    - insights-client
    - open-vm-tools
    - vim-enhanced
  # Partition configuration
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-Filesystem
  partitions:
    - min_size_gb: 50
      mountpoint: /
    - min_size_gb: 25
      mountpoint: /var
  platform: vmware
  # These values leverage CDN. Could point to Satellite, but not tested.
  # See https://developers.redhat.com/api-catalog/api/image-builder#schema-Subscription
  subscription:
    base_url: https://cdn.redhat.com/
    insights: true
    rhc: true
    server_url: subscription.rhsm.redhat.com
# List of VMs to provision using our image
vms:
    # Number of virtual CPUs
  - cpus: 2
    # RAM in GB
    memory: 8
    # VM hardware version in vCenter
    version: 21
    # The name of the VM in vCenter
    name: ceph0.lab.uc2.io
    # The OVA is deployed without NICs. These NICs are added to the VM after it is deployed.
    # For IPA integreation, the first NIC (i.e. network[0].ip) is used for DNS
    network:
      - broadcast: 172.16.10.255
        cidr: 24
        dns:
          - 172.16.10.2
        gateway: 172.16.10.254
        interface: eth0
        ip: 172.16.10.130
        netmask: 255.255.255.0
        search:
          - lab.uc2.io
        # Only `static` is currently supported.
        type: static
        vm_network: Lab Network - 10g
    vcenter:
      cluster: r440s
      datacenter: UC2
      datastore: r440-1-ssd-local
      folder: openshift/ceph
      guest_id: rhel9_64Guest
  - name: ceph1.lab.uc2.io
    network:
      - broadcast: 172.16.10.255
        cidr: 24
        dns:
          - 172.16.10.2
        gateway: 172.16.10.254
        interface: eth0
        ip: 172.16.10.131
        netmask: 255.255.255.0
        search:
          - lab.uc2.io
        type: static
    vcenter:
      cluster: r440s
      datacenter: UC2
      datastore: r440-2-ssd-local
      folder: openshift/ceph
      guest_id: rhel9_64Guest
  - name: ceph2.lab.uc2.io
    network:
      - broadcast: 172.16.10.255
        cidr: 24
        dns:
          - 172.16.10.2
        gateway: 172.16.10.254
        interface: eth0
        ip: 172.16.10.132
        netmask: 255.255.255.0
        search:
          - lab.uc2.io
        type: static
    vcenter:
      cluster: r440s
      datacenter: UC2
      datastore: r440-3-ssd-local
      folder: openshift/ceph
      guest_id: rhel9_64Guest
```

## Setting up Ansible

For development, a local Python virtual environment is used. The requirements.txt file in this repository contains all required dependencies (including Ansible).

### Create Python Virtual Environment

Create a Python virtual environment for the application.

```command
python -m venv /path/to/virtual/environment
```

Activate the virtual environment.

```command
. /path/to/virtual/environment/bin/activate
```

Upgrade pip and Install required Python dependences.

```command
pip install -U pip && pip install -U -r requirements.txt
```

### Create an Ansible Vault

Sensitive variables (like credentials for IdM or vCenter) should be stored in a vault. Create the vault as follows:

```command
ansible-vault create /path/to/vault.yaml
```

The following variables should be defined:

```yaml
activation_key: activation-key-name
activation_key_org: 1234567
ipa_hostname: idm.lab
ipa_password: changeme
ipa_username: admin
offline_token: eyJhbGciOiJ...
vcenter_username: administrator@vsphere.local
vcenter_password: changeme
vcenter_hostname: vcenter.lab
```

## Running create.yaml

To create an image and a set of virtual machines, run the following command:

```command
ansible-playbook -e @vault.yaml --ask-vault-pass -e @vars/properties.yaml spawn.yaml
```

`properties.yaml` is a file you create (see [Defining VM and Image Properties](#defining-vm-and-image-properties) above).

If you wish to reuse a previously generated OVA, you can add `--skip-tags image` and pass the ova_local_path variable as an extra var.

```command
ansible-playbook -e @vault.yaml --ask-vault-pass -e @vars/properties.yaml spawn.yaml --skip-tags image -e ova_local_path=/path/to/image.ova
```
