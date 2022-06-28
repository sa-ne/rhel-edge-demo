# RHEL for Edge Demo

This repository contains a set of playbooks you can use to build out the infrastructure to run a RHEL for Edge (RFE) demo. The playbooks will enable you to:

* Leverage the hosted Image Builder on [console.redhat.com](https://console.redhat.com/insights/image-builder) to build a RHEL 9 VM with Image Builder pre-installed.
* Deploy Image Builder VM on KVM or AWS
* Deploy Quay on Image Builder VM to host RHEL for Edge (RFE) OSTree containers.

The following use cases are highlighted:

* Leverage the hosted Image Builder API to build and download RHEL virtual machines.
* Modify downloaded VM image (set root password, resize disk, etc).
* Leverage Blueprints to create RFE content in Image Builder.
* Lifecycle RFE content in Red Hat Quay.
* Deploy RFE guests.
* Patch RFE guests.
* Use Ansible to modify RFE guests.
* Test Greenboot with RFE guests.

## Architecture

All of the components to build and manage RFE content (including Image Builder, Quay and Apache) are hosted on a RHEL 9 virtual machine. The playbooks support the deployment of this virtual machine on AWS or locally using KVM. The RFE deployments in this demo are ISO based and require KVM.

![Architecture](/images/architecture.png)

## Requirements

To leverage the automation in this demo you need to bring the following:

RHEL/Fedora KVM host with at least 4 vCPUs, 16GB RAM and 40G of available storage. We will also need the following packages installed:

* ansible (tested on 2.12.6)
* guestfs-tools
* python3-botocore
* python3-boto3

### Offline Token for Red Hat API

To make API calls against the hosted Image Builder API we will need an offline token. You can generate one [here](https://access.redhat.com/management/api).

### Simple Content Access

You will also need to make sure the Simple content access for Red Hat Subscription Management feature is enabled in the customer portal [here](https://access.redhat.com/management).

## Setting up Environment

Before beginning, you will need to clone this git repository as follows:

```shell
git clone https://github.com/sa-ne/rhel-edge-demo.git
```

The configuration options for the deployment are stored in two files and differ slightly depending on the platform you are targeting (either KVM or AWS). Sensitive variables (keys, credentials, etc.) are stored in an Ansible vault and more general configuration options are stored in a variables file. See below for additional details.

### Install Required Collections

The playbooks in this repository make use of various Ansible Collections. To install them, run the following command:

```shell
ansible-galaxy collection install -r collections/requirements.yaml
```

### Create Ansible Vault

As previously mentioned, some variables contain sensitive data so we will store them in an Ansible Vault. You can create the vault in the `local/` directory at the root of the repository as follows:

```shell
ansible-vault create local/vault.yaml
```

We will need to configure the following variables that are common for both AWS and KVM based deployments.

|Variable|Description|
|:---|:---|
|activation_key|Activation key used to register the system. Create one [here](https://access.redhat.com/management/activation_keys).|
|activation_key_org|Organization ID tied to activation key. This can be found at the top of [Activation Keys](https://access.redhat.com/management/activation_keys) section on the customer portal.|
|hib_root_password|Root password for the Image Builder VM.|
|quay_rfe_password|Password for the Quay admin account.|
|quay_rfe_username|Username for the Quay admin account.|
|redhat_api_offline_token|Token used to authenticate with Red Hat APIs in the Hybrid Cloud Console. Generate one [here](https://access.redhat.com/management/api).|

When finished, your vault.yaml should look similar to the example below:

```yaml
activation_key: my-activation-key
activation_key_org: 1234567
hib_root_password: $3cur3
quay_rfe_password: $3cur3
quay_rfe_username: rfe
redhat_api_offline_token: |-
  <long string>
```

## Deploying Image Builder on KVM

These playbooks assume KVM is running _locally_ (i.e. the virtual machine will be deployed to localhost).

### Compose Image Builder VM for KVM

We will need to add some KVM specific variables to `vars/config.yaml` and `local/vault.yaml`. For our vault, the following variable should be added:

|Variable|Description|
|:---|:---|
|hib_libvirt_ssh_public_key_path|Path of SSH key to associate with instance.|

When finished, the variables in your vault should look similar to the following:

```yaml
activation_key: my-activation-key
activation_key_org: 1234567
hib_libvirt_ssh_public_key_path: /home/user/.ssh/id_rsa.pub
hib_root_password: $3cur3
quay_rfe_password: $3cur3
quay_rfe_username: rfe
redhat_api_offline_token: |-
  <long string>
```

For `vars/config.yaml`, make sure the following variables are updated as necessary. Generally speaking, the default vaules should apply.

|Variable|Description|
|:---|:---|
|hib_libvirt_disk_path|Path to libvirt storage.|
|hib_libvirt_lease_file|Path to libvirt lease file.|
|hib_libvirt_ram|Amount of RAM for Image Builder (GB).|
|hib_libvirt_vcpu|Number of vCPUs for Image Builder.|
|hib_name|Name of Image Builder instance.|
|hib_rhel_distribution|Distribution to use for Image Builder instance.|
|hib_root_filesystem_size|Size of Image Builder disk (GB).|
|hib_libvirt_qemu_path|Path to QEMU configuration.|
|platform|Target platform for Image Builder deployment.|

When finished, the variables in `vars/config.yaml should look similar to the following:

```yaml
hib_libvirt_disk_path: /var/lib/libvirt/images
hib_libvirt_lease_file: /var/lib/libvirt/dnsmasq/virbr0.status
hib_libvirt_ram: 8
hib_libvirt_vcpu: 4
hib_name: rhel90-image-builder
hib_rhel_distribution: rhel-90
hib_root_filesystem_size: 25
hid_libvirt_qemu_path: /etc/libvirt/qemu
platform: libvirt
```

Run the playbook as follows:

```shell
ansible-playbook \
  --ask-vault-pass \
  -e @local/vault.yaml \
  01-compose-image-builder.yaml
```

### Deploy Image Builder VM on KVM

For AWS specific composes, the AMI id will be outputted to the playbook results. The ID will also be saved in /tmp with the format `<hib_name>-<timestamp>-ami-id`.

*_NOTE: The compose process takes about 15 minutes to complete._*

## Deploy Image Builder VM

This playbook will deploy/configure the Image Builder VM. If `platform` is set to `aws` a Security Group and EC2 Instance will get created. For `libvirt` based deployments the playbooks will leverage `virt-customize` to modify the image. With either platform selected, after the Image Builder instance comes up the playbooks will configure and enable services and deploy Quay.

We will need to pass credentials for the Image Builder instance to the `ansible-playbook` command. For AWS based deployments, the user will be `ec2-user`. Additionally, you will need to pass the `hib_aws_ami` variable. For libvirt based deployments, the user will be root. In either case, the public SSH key defined in `vars/config.yaml` will be used to authenticate.

Run the playbook as follows:

```shell
ansible-playbook \
  --ask-vault-pass \
  -e @local/vault.yaml \
  -u root \
  --private-key /home/chris/.ssh/id_rsa.pub \
  02-deploy-and-configure-image-builder.yaml
```

## Deploying Image Builder on KVM

### Compose Image Builder VM for AWS

## Demo

## Cleanup

To remove the Image Builder instance, run the following playbook:

```shell
ansible-playbook \
  --ask-vault-pass
  -e @local/vault.yaml \
  05-delete-demo.yaml
```
