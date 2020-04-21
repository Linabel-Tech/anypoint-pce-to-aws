## Create Anypoint Private Cloud Resources on Amazon Web Services (AWS)
``` 
How-to guide to create resources required to install Anypoint PCE on AWS.
Anypoint PCE supports 3-node and 6-node configuration in a prod env on AWS.
```
The **Anypoint Platform Private Cloud Edition** provides the following application components available in the online version of Anypoint Platform:
* **Runtime Manager:** enables you to track and manage your deployed Mule applications
* **API Manager:** enables you to manage your registered APIs
* **Anypoint Exchange:** enables your organization to **share various assets** that can be **reused across integration projects**.

- You can run and manage Mule applications on your local servers.
- Anypoint Platform PCE uses Docker and Kubernetes to provide built-in high availability and scalability.
- You can install the platform without understanding Docker or Kubernetes.


**Prerequisites**:

* AWS keys with **EC2FullAccess** and **S3FullAccess permissions**.


When creating your AWS environment, the following resources are created:

| AWS Resource | Number required (3-node) | Number required (6-node)
| ------------- |:-------------:| -----:|
| m4.2xlarge | 3 | 6
| root disk @ 500 iops | 3 | 6
| EBS volumes @ 1500 iops | 6 | 12
| EBS volume @ 3000 iops | 6 | 12
| Amazon ELB | 1 | 1
| t2.medium | 1 | 1


## Run the AWS Provisioner

MuleSoft provides a **Docker image** that you can use to provision the resources for your AWS account.

#### 1.) Create an initial instance (**t2.small**) on your AWS account on any VPC with internet access.

This instance is used to run the actual provisioning of the cluster. You must use an **AMI** that has Docker installed by default or you must manually install Docker after creating the AWS instance.

You can also run provisioner remotely from any other machine with Docker and internet access.

#### 2.) Download the provisioner [Docker image](https://onprem-standalone-installers.s3.amazonaws.com/pce2.0.1-aws-provisioner.tar.gz?Expires=1709789111&AWSAccessKeyId=AKIAI6LBJNQFR2V3CQWQ&Signature=4aQ4wJNuRDzCiz%2BF6iKfwZIUyPM%3D)

#### 3.) Copy the provisioner Docker image to the instance via SCP
```
scp -i <guest>.pem ~/Downloads/pce-2.0-aws-provisioner.gz ec2-user@18.221.155.87:/home/ec2-user
```
#### 4.) SSH into the instance
```
ssh -i 'anypoint.pem' ec2-user@18.221.155.87
```
#### 5.) Create the var file (pce.env) with the environment details using the template below:


| Name | Description
| ------------- |:-------------:|
| AWS_ACCESS_KEY_ID | Specifies the AWS access ID Terraform uses to connect to your AWS account
| AWS_SECRET_ACCESS_KEY | Specifies the AWS access key Terraform uses to connect to your AWS account
| AWS_KEY_NAME | Specifies the AWS SSH key. Do not include the .pem extension.
| AWS_REGION | Specifies the AWS region where Terraform creates the cluster. For example: `us-east-2`. Must have the same value as AWS_DEFAULT_REGION.
| AWS_DEFAULT_REGION | Same as above. Must have the same value as AWS_REGION
| TF_VAR_ssh_user=<SSH_USER> | Specifies the ssh user. For example: `ec2-user`, `centos`, etc)
| CLOUD_PROVIDER | Must be `aws`
| CLUSTER_TYPE | Must be `pce`
| TELEKUBE_CLUSTER_NAME | Specifies the name of the cluster and the corresponding AWS resources
| TELEKUBE_NODE_PROFILE_COUNT_node | Specifies the number of nodes in your cluster. Possible values are `3`, and `6`
| TELEKUBE_NODE_PROFILE_INSTANCE_TYPE_node | Must be `m4.2xlarge`
| TELEKUBE_NODE_PROFILES | Must be set to `node`
| TF_VAR_high_performance_disks | Must be set to `true` for production environments
| TF_VAR_disable_outbound_traffic | Disables outside internet access using iptables rules. Defaults to `true`.


You can also include the following set of optional environment variables:

| Name | Description
| ------------- |:-------------:|
| TF_VAR_ami_name | Enables you to specify the name of the AMI to be used for the instances. Use the AMI name and NOT the AMI Id. If you do not set this variable, the provisioner uses the folowing AMI by default: `RHEL-7.4_HVM_GA-20170808-x86_64-2-Hourly2-GP2`
| TF_VAR_monitoring=<true or false> | If true, it will create basic instance alarms in Cloudwatch.
| TF_VAR_use_bastion=<true or false> | If true, it will create a small instance (using an ASG) in a public subnet as a jumpbox and associate a public IP address, and launch the cluster instances in the private subnets
| TF_VAR_internal=<true or false> | If true the cluster instances will be launched in private subnets and will not have public ips associated with them. Also, the provisioned load balancer will be internal only
| AWS_VPC_ID=<VPC_ID> | Specifies an existing VPC. It should already have an internet gateway attached to it. Subnets will still be provisioned within the existing VPC
| AWS_VPC_CIDR=<VPC_CIDR> | Enables you to pass a preferred CIDR for the new VPC that will be provisioned. (Requires not to pass a VPC_ID)
| TF_VAR_subnets=<PRIVATE_SUBNETS_CIDRs> | Enables you to pass the preferred CIDRs blocks for the new subnets that will be provisioned. You must pass exactly the minimum num	ber between the number of AZs of the region and the number of nodes you selected for your cluster. For example: `'["10.1.0.0/24", "10.1.1.0/24", "10.1.2.0/24"]'`
| TF_VAR_public_subnets=<PUBLIC_SUBNETS_CIDRs> | Enables you to pass the preferred CIDRs blocks for the new subnets that will be provisioned. You must pass exactly the minimum num	ber between the number of AZs of the region and the number of nodes you selected for your cluster. For example: `'["10.1.3.0/24", "10.1.4.0/24", "10.1.5.0/24"]'`
| TF_VAR_role_tag_value=<AWS_TAG_VALUE_FOR_ROLE_LABEL> | Enables you to specify a ROLE tag value to be applied to all AWS resources.


#### 6.) Load the provisioner Docker image into the local Docker registry:
```
docker load -i pce2.0-aws-provisioner.gz
```
#### 7.) Record the image ID after running the `docker load` command. This is required in the following step.

#### 8.) Perform a dry-run
```
docker run --rm --env-file pce.env IMAGE_ID dry-run
```
#### 9.) Run the provisioner
```
docker run --rm --env-file pce.env IMAGE_ID cluster-provision
```

After the provisioner runs successfully, it displays information about your environment including **IP addresses, DNS name of the load balancer, etc.**

#### 10.) Verify that the provisioning script ran successfully by checking the existence of `/var/lib/bootstrap_complete` on the instances.

## Open Port 61009 Before Installation (Optional)

If you are installing Anypoint Private Cloud Edition using the GUI-based installer, you must enable this port before running the installer. You must open this port to the world in the cluster's security group before running the installer using AWS Web Console.

## Install Anypoint Platform Private Cloud Edition

After provisioning resources in your AWS environment and uploading the installer to one of the nodes, install Anypoint Platform Private Cloud Edition using one of the installers:

* xref:install-installer.adoc[To Install Anypoint Private Cloud using the GUI Installer]
* xref:install-auto-install.adoc[To Install Anypoint Private Cloud using the Command Line Installer]

## Disabling Port 61009 After Installation

After installation is finished you can close port 61009 in the cluster security group using the AWS Web console.

## Troubleshooting

**Error:** `403 Forbidden`
Check that your keys policies are not denying access to any EC2 or S3 resource. You should be able to run the following basic command using `AWS CLI` with your keys:
```
aws sts get-caller-identity
```

**Error:** `Limit excedded`

The AWS account might have some **limits set on the amount of the resources** you can create. **Delete some unused resources** or **request a limit increason from AWS**.

**Error:** `AMI not found`

Make sure you are using the **AMI name** as the value for the environment variable and **not** the ID.
