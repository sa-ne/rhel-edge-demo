# RHEL for Edge Demo

This repository contains a set of playbooks you can use to build out the infrastructure to run a RHEL for Edge demo. The playbooks will enable you to:

* Leverage the hosted Image Builder on [console.redhat.com](https://console.redhat.com/insights/image-builder) to build a RHEL 9 VM with Image Builder pre-installed.
* Download Image Builder VM and deploy using KVM (or share an AMI with an AWS account)
* Start and configure Image Builder VM
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

## Requirements

To leverage the automation in this demo you need to bring the following:

RHEL/Fedora KVM host with at least 4 vCPUs, 16GB RAM and 40G of available storage. We will also need the following packages installed:

* ansible
* guestfs-tools
* python3-botocore
* python3-boto3

### Simple Content Access

You will also need to make sure the Simple content access for Red Hat Subscription Management feature is enable in the customer portal [here](https://access.redhat.com/management).

## Prerequisites

Before beginning, you will need to clone this git repository as follows:

```shell
git clone https://github.com/sa-ne/rhel-edge-demo.git
```

Next we will need to create an Ansible vault with some variables:

|Variable|Required|Description|
|:---|:---|:---|
|activation_key|Yes|Activation key used to register the system. Create one [here](https://access.redhat.com/management/activation_keys).|
|activation_key_org|Yes|Organization ID tied to activation key. This can be found at the top of [Activation Keys](https://access.redhat.com/management/activation_keys) section on the customer portal.|
|aws_access_key|AWS Only|Access key for IAM user.|
|aws_secret_key|AWS Only|Secret key for IAM user.|
|hib_aws_account_id|AWS Only|AWS Account ID.|
|hib_aws_key_name|AWS Only|SSH key to associate with instance.|
|hib_aws_subnet_id|AWS Only|Subnet ID to associate with instance.|
|hib_aws_vpc_id|AWS Only|VPC ID to associate with instance.|
|hib_libvirt_ssh_public_key_path|KVM Only|SSH key to associate with instance.|
|hib_root_password|Yes|Root password for the Image Builder VM.|
|quay_rfe_password|Yes|Password for the Quay admin account.|
|quay_rfe_username|Yes|Username for the Quay admin account.|
|redhat_api_offline_token|Yes|Token used to authenticate with Red Hat APIs in the Hybrid Cloud Console. Generate one [here](https://access.redhat.com/management/api).|

### Create Ansible Vault

Most of the variables referenced above contain sensitive data so we will store them in an Ansible Vault. You can create the vault in the `local/` directory at the root of the repository as follows:

```shell
ansible-vault create local/vault.yaml
```

When finished, your vault.yaml should look similar to the example below:

```yaml
activation_key: my-activation-key
activation_key_org: 1234567
aws_access_key: AKIASDFGHJKLO1234FGHY
aws_secret_key: neWvTeGYdiR5y2DaJZf9127hv9d3fk1EtHiakl4DX
hib_aws_account_id: 12345678901
hib_aws_key_name: my-key-name
hib_aws_subnet_id: subnet-ak3jd81mkvfu23jfj
hib_aws_vpc_id: vpc-ak3jd81lwvfu23jfq
hib_libvirt_ssh_public_key_path: /home/chris/.ssh/id_rsa.pub
hib_root_password: $3cur3
quay_rfe_password: $3cur3
quay_rfe_username: rfe
redhat_api_offline_token: |-
  <long string>
```

### Install Required Collections

The playbooks in this repository make use of various Ansible Collections. To install them, run the following command:

```shell
ansible-galaxy collection install -r collections/requirements.yaml
```

### Configuration Options

Several variables to control the deployment are included in `vars/config.yaml`.

|Variable|Description|
|:---|:---|
|hib_aws_device_name|Partition to use for /.|
|hib_aws_instance_initiated_shutdown_behavior|Shutdown behavior for instance.|
|hib_aws_instance_type|AWS instance type.|
|hib_aws_region|AWS region for deployment.|
|hib_aws_security_group_descrition|Description for generated AWS security group.|
|hib_aws_security_group_name|Name for generated AWS security group.|
|hib_aws_volume_iops|IOPS for Image Builder disk.|
|hib_aws_volume_type|Type of EBS volume for Image Builder disk.|
|hib_libvirt_disk_path|Path to libvirt storage.|
|hib_libvirt_lease_file|Path to libvirt lease file.|
|hib_libvirt_ram|Amount of RAM for Image Builder (GB).|
|hib_libvirt_vcpu|Number of vCPUs for Image Builder.|
|hib_name|Name of Image Builder instance.|
|hib_rhel_distribution|Distribution to use for Image Builder instance.|
|hib_root_filesystem_size|Size of Image Builder disk (GB).|
|hid_libvirt_qemu_path|Path to QEMU configuration.|
|platform|Target platform for Image Builder deployment.|

For the most part the default values in `vars/config.yaml` can be used. Just be sure to adjust the `platform` and set it to either `aws` or `libvirt`.

## Compose Image Builder VM

This playbook will compose a RHEL 9 image with Image Builder, Quay and other tooling preinstalled. If the `platform` variable is set to `aws` the hosted Image Builder service will share the AMI privately with your AWS account. For `libvirt` based deployments, the playbook will download the generated QCOW2 image locally in your temp directory.

Run the playbook as follows:

```shell
ansible-playbook \
  --ask-vault-pass \
  -e @local/vault.yaml \
  01-compose-image-builder.yaml
```

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

## Cleanup

To remove the Image Builder instance, run the following playbook:

```shell
ansible-playbook \
  --ask-vault-pass
  -e @local/vault.yaml \
  05-delete-demo.yaml
```
