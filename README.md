# c44-demo-ansible-terraform

Quick demo of how to integrate Ansible and Terraform to manage AWS resources

## Resources

The following AWS resources will be created:

* VPC
* Subnets (Private and Public)
  * `Private and Public` subnets have ACLs to `allow ALL ingress/egress traffic from/to 0.0.0.0/0` (not recommended, but ok for this demo)
* Internet Gateway
* Route Table
* S3 Bucket
* SSH Keypair
  * This `keypair` will be `stored in the S3 bucket` created by the this playbook
* Security Groups for SSH access
  * `Allow Ingress` traffic on `port TCP/22` from `Office` and `VPC subnets`
  * `Allow Egress` traffic on `ALL` ports to `0.0.0.0/0` (not recommended, but ok for this demo)
* EC2 Instances
  * `Public IP` addresses, which are not EIPs `will be assigned` (not recommended, but ok for this demo)

Once initialization is completed, the Terraform state is persisted into an S3 bucket. See `defaults/main.yml`, section `terraform.backend` for more information.

> Terraform must store state about your managed infrastructure and configuration. This state is used by Terraform to map real world resources to your configuration, keep track of metadata, and to improve performance for large infrastructures.

Source [Terraform State](https://www.terraform.io/docs/state/index.html)

## Requirements

### Software

* [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) >= 2.8.x
* [aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) >= 1.16.260
* [curl](https://curl.haxx.se/) >= 7.64.1
* [jq](https://stedolan.github.io/jq/download/) >= 1.6
* [terraform](https://www.terraform.io/downloads.html) >= v0.12.25

### AWS Credentials

Before proceeding, make sure an IAM User is created, with permissions required to create and manage the AWS resources provisioned by this playbook.

Click [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) for more details on *Managing Access Keys for IAM Users*

As soon as an Access Key is created, configure your local environment as folllow:

```bash
$ export AWS_DEFAULT_REGION=ca-central-1
$ export AWS_ACCESS_KEY_ID=__AWS_ACCESS_KEY_ID__
$ export AWS_SECRET_ACCESS_KEY=__AWS_SECRET_ACCESS_KEY__
```

### Infrastructure Options (variables)

Ansible can be used as an orchestration tool to prepare all requirements used by Terraform. From SSH keypais to performing the actual plan, apply and destroy Terraform actions, Ansible can be used to *glue* all the pieces together prior to proceed with the provisioning the of the beforementioned AWS resources.

We leverage as well, the Templating mechanism available in Ansible, which is provided by the Jinja2 Templating Language.

> Jinja is a modern and designer-friendly templating language for Python, modelled after Django’s templates. It is fast, widely used and secure with the optional sandboxed template execution environment:

Source [Jinja](https://jinja.palletsprojects.com/en/2.11.x/)

For that reason:

* Environment configuration and infrastructure options are declared in the `defaults/main.yml` file
* Make sure you revisit this file prior to initiate the provisioning of the infrastructure

## Main Playbook

The main playbook is called `demo.yml` and can be found in the root of this repository. This playbook will act as an entrypoint and allow us to:

#### Provision the infrastructure

```yaml
- name: "[INFO] DEMO FOR TASKS EXECUTED LOCALLY"
  hosts: localhost

...

tasks:

  - include_tasks: tasks/init.yml
  - include_tasks: tasks/provision.yml
```

#### Configure the Infrastructure

```yaml
- name: "[INFO] DEMO FOR TASKS EXECUTED ON REMOTE HOSTS"
  hosts: "{{ inventory.name | default('all') }}"

...

tasks:
  - include_tasks: tasks/configure.yml
```

## Terraform Plan

> The terraform plan command is used to create an execution plan. Terraform performs a refresh, unless explicitly disabled, and then determines what actions are necessary to achieve the desired state specified in the configuration files.

Source [Terraform: command plan](https://www.terraform.io/docs/commands/plan.html)

```bash
$ ansible-playbook demo.yml \
    -e 'project.id'=c44 \
    -e enable_plan=true
```

## Terraform Apply

> The terraform apply command is used to apply the changes required to reach the desired state of the configuration, or the pre-determined set of actions generated by a terraform plan execution plan.

Source [Terraform: command apply](https://www.terraform.io/docs/commands/apply.html)

```bash
$ ansible-playbook demo.yml \
    -e 'project.id'=c44 \
    -e enable_apply=true \
    -e enable_inventory=true \
    -e enable_configure=true
```

### Reconfigure EC2 instances

The whole objective of integrating Ansible with Terraform is that, as soon as the infrastructure is provisioned, you then have an option to configure your servers. With Ansible, you could pass a list of hosts (coming from a static or dynamic inventory) and execute tasks against these hosts.

Since Terraform knows our infrastructure from A to Z, we have the option to fetch the EC2 instances IPs and prepare a dynamic inventory that will be used by Ansible.

> You can run the example as many times as you required, since it will never create or delete resources (they are managed by Terraform).

The example below will prepare a dynamic inventory, based on hosts found in your Terraform State file and then execute any tasks declared in the `tasks/configure.yml` file:

```bash
$ ansible-playbook demo.yml \
    -e 'project.id'=c44 \
    -e enable_inventory=true \
    -e enable_configure=true
```

## Terraform Destroy

> The terraform destroy command is used to destroy the Terraform-managed infrastructure.

Source [Terraform: command destroy](https://www.terraform.io/docs/commands/destroy.html)

```bash
$ ansible-playbook demo.yml \
    -e 'project.id'=c44 \
    -e enable_destroy=true
```