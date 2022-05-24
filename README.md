# RHEL for Edge Demo

This repository contains a set of playbooks you can use to build out the infrastructure to run a RHEL for Edge demo. The playbooks will enable you to:

* Leverage the hosted Image Builder on (console.redhat.com)[https://console.redhat.com/beta/insights/image-builder] to build a RHEL 8.6 VM with Image Builder pre-installed.
* Download Image Builder VM and deploy to supported targets (currently KVM).
* Start and configure Image Builder VM
* Deploy Quay on Image Builder VM to host RHEL for Edge (RFE) OSTree containers.

The following use cases are highlighted:

* Leverage the hosted Image Builder API to build and download RHEL virtual machines.
* Modify downloaded VM image (set root password, resize disk).
* Leverage Blueprints to create RFE content in Image Builder.
* Lifecycle RFE content in Red Hat Quay.
* Deploy RFE guests.
* Patch RFE guests.
* Use Ansible to modify RFE guests.
* Test Greenboot with RFE guests.

## Requirements

To leverage the automation in this demo you need to bring the following:

RHEL/Fedora KVM host with at least 4 vCPUs, 16GB RAM and 40G of available storage. We will also need the following packages installed:

* ansible
* guestfs-tools

### Simple Content Access

You will also need to make sure the Simple content access for Red Hat Subscription Management feature is enable in the customer portal (here)[https://access.redhat.com/management].

## Prerequisites

Before beginning, you will need to clone this git repository as follows:

```shell
git clone https://github.com/sa-ne/rhel-edge-demo.git
```

Next we will need to create an Ansible vault with some variables:

|Variable|Description|
|:---|:---|
|activation_key|Activation key used to register the system. Create one [here](https://access.redhat.com/management/activation_keys).|
|activation_key_org|Organization ID tied to activation key. This can be found at the top of (Activation Keys)[https://access.redhat.com/management/activation_keys] section on the customer portal.|
|hib_root_password|Root password for the Image Builder VM.|
|redhat_api_offline_token|Token used to authenticate with Red Hat APIs in the Hybrid Cloud Console. Generate one (here)[https://access.redhat.com/management/api].|
|quay_password|Password for the Quay admin account.|
|quay_username|Username for the Quay admin account.|

### Create Ansible Vault

Most of the variables referenced above contain sensitive data so we will store them in an Ansible Vault. You can create the vault in the `local/` directory at the root of the repository as follows:

```shell
ansible-vault create local/vault.yaml
```

When finished, your vault.yaml should look similar to the example below:

```yaml
activation_key: image-builder-demo
activation_key_org: 1234567
hib_root_password: s3cr3t
redhat_api_offline_token: |-
  averylongstring
quay_password: s3cr3t
quay_username: edge
```

### Install Required Collections

The playbooks in this repository make use of various Ansible Collections. To install them, run the following command:

```shell
ansible-galaxy collection install -r collections/requirements.yaml
```

## Compose and Download Image Builder VM for KVM

This playbook will compose and download a RHEL 8.6 qcow2 image with Image Builder preinstalled. The following default values are used (these can be overwritten on the command line):

```yaml
hib_name: rhel86-image-builder
hib_root_filesystem_size: 25
```

|Variable|Description|
|:---|:---|
|hib_name|Name of the compose as well as the name of the deployed Image Builder VM.|
|hib_root_filesystem_size|Size of the Image Builder VM root filesystem in Gigabytes (GB).|

We will also need to include the variables defined in our vault. To run the playbook, execute the following:

```shell
ansible-playbook \
  --ask-vault-pass \
  -e @local/vault.yaml \
  -e hib_name=image-builder-demo \
  -e hib_root_filesystem_size=50 \
  01-compose-image-builder.yaml
```

This example overrides the `hib_name` default and changes the value to `image-builder-demo`. It also overrides `hib_root_filesystem_size` to be 50G.

*_NOTE: The compose process takes about 15 minutes to complete._*

