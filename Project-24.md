## __BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM__

In this extensive project, our primary emphasis will be on the practical real-world implementation of Kubernetes deployment and management. This involves a series of tasks geared toward guaranteeing the resilience and scalability of our system. The key objectives include:

- I plan to employ Terraform to establish a Kubernetes EKS cluster and dynamically integrate adaptable worker nodes into the system. 

- Furthermore, I intend to deploy multiple applications using __HELM__, leveraging Kubernetes objects in conjunction with Helm. This approach includes dynamic provisioning of volumes to enable stateful pods.

- Finally, I will execute the deployment process via CI/CD using Jenkins.

__Building EKS with Terraform__

Create a directory on your local machine - __eks__ and create an s3 bucket

`$ mkdir eks && cd eks`

`$ aws s3api create-bucket --bucket eks-terraform-deploy --region us-east-1`

![](./images/ekst.PNG)
![](./images/ekstt.PNG)

Create a file – __backend.tf__ Task for you, ensure the backend is configured for remote state.

Create a file – __network.tf__ and provision Elastic IP for Nat Gateway, VPC, Private and public subnets.

```
# reserve Elastic IP to be used in our NAT gateway
resource "aws_eip" "nat_gw_elastic_ip" {
  vpc = true

  tags = {
    Name            = "${var.cluster_name}-nat-eip"
    iac_environment = var.iac-env-tag
  }
}


module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "${var.name_prefix}-vpc"
  cidr = var.main_network_block
  azs  = data.aws_availability_zones.available_azs.names

  private_subnets = [
    # this loop will create a one-line list as ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", ...]
    # with a length depending on how many Zones are available
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) - 1)
  ]

  public_subnets = [
    # this loop will create a one-line list as ["10.0.128.0/20", "10.0.144.0/20", "10.0.160.0/20", ...]
    # with a length depending on how many Zones are available
    # there is a zone Offset variable, to make sure no collisions are present with private subnet blocks
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) + var.zone_offset - 1)
  ]

  # Enable single NAT Gateway - This is cost effective
  # WARNING: this could create a single point of failure, since we are creating a NAT Gateway in one AZ only
  # feel free to change these options if you need to ensure full Availability without the need of running 'terraform apply'
  # reference: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/2.44.0#nat-gateway-scenarios
  enable_nat_gateway     = true
  single_nat_gateway     = true
  one_nat_gateway_per_az = false
  enable_dns_hostnames   = true
  reuse_nat_ips          = true
  external_nat_ip_ids    = [aws_eip.nat_gw_elastic_ip.id]

  # Add VPC/Subnet tags required by EKS
  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    iac_environment                             = var.iac-env-tag
  }
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
    iac_environment                             = var.iac-env-tag
  }
  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
    iac_environment                             = var.iac-env-tag
  }
}
```

The tags added to the subnets is very important. The Kubernetes Cloud Controller Manager (cloud-controller-manager) and AWS Load Balancer Controller (aws-load-balancer-controller) needs to identify the cluster’s. To do that, it querries the cluster’s subnets by using the tags as a filter.

For public and private subnets that use load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/cluster/cluster-name
Value: shared
```

For private subnets that use internal load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/role/internal-elb
Value: 1
```

For public subnets that use internal load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/role/elb
Value: 1
```
Create a file – __variables.tf__

```
# create some variables
variable "cluster_name" {
  type        = string
  description = "EKS cluster name"
}
variable "iac-env-tag" {
  type        = string
  description = "AWS tag to indicate environment name of each infrastructure object."
}
variable "name_prefix" {
  type        = string
  description = "Prefix to be used on each infrastructure object Name created in AWS."
}
variable "main_network_block" {
  type        = string
  description = "Base CIDR block to be used in our VPC."
}
variable "subnet_prefix_extension" {
  type        = number
  description = "CIDR block bits extension to calculate CIDR blocks of each subnetwork."
}
variable "zone_offset" {
  type        = number
  description = "CIDR block bits extension offset to calculate Public subnets, avoiding collisions with Private subnets."
}
variable "admin_users" {
  type        = list(string)
  description = "List of Kubernetes admins."
}
variable "developer_users" {
  type        = list(string)
  description = "List of Kubernetes developers."
}
variable "asg_instance_types" {
  description = "List of EC2 instance machine types to be used in EKS."
}
variable "autoscaling_minimum_size_by_az" {
  type        = number
  description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_maximum_size_by_az" {
  type        = number
  description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."
}

variable "autoscaling_average_cpu" {
  type        = number
  description = "number of cpu for autoscaling"

}
```
Create a file – __data.tf__ – lists available AZs in the region

```
# get all available AZs in our region
data "aws_availability_zones" "available_azs" {
state = "available"
}
data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
```
Create a file – __eks.tf__ and provision EKS cluster

```
module "eks_cluster" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 18.0"
  cluster_name    = var.cluster_name
  cluster_version = "1.22"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access = true

  # Self Managed Node Group(s)
  self_managed_node_group_defaults = {
    instance_type                          = var.asg_instance_types[0]
    update_launch_template_default_version = true
  }
  self_managed_node_groups = local.self_managed_node_groups

  # aws-auth configmap
  create_aws_auth_configmap = true
  manage_aws_auth_configmap = true
  aws_auth_users = concat(local.admin_user_map_users, local.developer_user_map_users)
  tags = {
    Environment = "prod"
    Terraform   = "true"
  }
}
```
To establish local variables in Terraform, you should create a file named __locals.tf__. Terraform prohibits the direct assignment of variables to other variables, a restriction designed to prevent unnecessary code duplication. To maintain the __"Don't Repeat Yourself" (DRY)__ principle in the Terraform code, it's advisable to utilize locals instead.

```
# render Admin & Developer users list with the structure required by EKS module
locals {
  admin_user_map_users = [
    for admin_user in var.admin_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${admin_user}"
      username = admin_user
      groups   = ["system:masters"]
    }
  ]
  developer_user_map_users = [
    for developer_user in var.developer_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${developer_user}"
      username = developer_user
      groups   = ["${var.name_prefix}-developers"]
    }
  ]

  self_managed_node_groups = {
    worker_group1 = {
      name = "${var.cluster_name}-wg"

      min_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      desired_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      max_size  = var.autoscaling_maximum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      instance_type = var.asg_instance_types[0].instance_type

      bootstrap_extra_args = "--kubelet-extra-args '--node-labels=node.kubernetes.io/lifecycle=spot'"

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            delete_on_termination = true
            encrypted             = false
            volume_size           = 10
            volume_type           = "gp2"
          }
        }
      }

      use_mixed_instances_policy = true
      mixed_instances_policy = {
        instances_distribution = {
          spot_instance_pools = 4
        }

        override = var.asg_instance_types
      }
    }
  }
}
```
Create a file – __variables.tfvars__ to set values for variables.

```
cluster_name            = "dybran-tooling-app"
iac_environment_tag     = "development"
name_prefix             = "dybran-tooling-app"
main_network_block      = "10.0.0.0/16"
subnet_prefix_extension = 4
zone_offset             = 8

# Ensure that these users already exist in AWS IAM. Another approach is that you can introduce an iam.tf file to manage users separately, get the data source and interpolate their ARN.
admin_users                    = ["peter", "solomon"]
developer_users                = ["leke", "david"]
asg_instance_types             = [ { instance_type = "t3.small" }, { instance_type = "t2.small" }, ]
autoscaling_minimum_size_by_az = 1
autoscaling_maximum_size_by_az = 10
autoscaling_average_cpu        = 30
```

Create file – __provider.tf__

```
provider "aws" {
  region = "us-west-1"
}

provider "random" {
}
```

__See Terraform code [here](https://github.com/dybran/Project-24/tree/main/terraform-eks)__

Open Powershell and run as administrator and install terraform using __chocolatey__ package manager

`$ choco install terraform -y`

Run

`$ terraform init`

`$ Terraform plan`

![](./images/t1.PNG)
![](./images/t2.PNG)

Create the resources

`$ Terraform apply`

At some point, it fails.

![](./images/error2.PNG)

That is because for us to connect to the cluster using the kubeconfig, Terraform needs to be able to connect and set the credentials correctly.

To fix this problem

Append to the file __data.tf__

```
# get EKS cluster info to configure Kubernetes and Helm providers
data "aws_eks_cluster" "cluster" {
  name = module.eks_cluster.cluster_id
}
data "aws_eks_cluster_auth" "cluster" {
  name = module.eks_cluster.cluster_id
}
```
![](./images/aaaq.PNG)

Append to the file __provider.tf__

```
# get EKS authentication for being able to manage k8s objects from terraform
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
```
![](./images/aaaq.PNG)


Run again

`$ terraform init`

`$ Terraform plan`

![](./images/auth.PNG)

Run

`$ terraform apply`

![](./images/qw.PNG)
![](./images/rf.PNG)

Create kubeconfig file using awscli.

`$ aws eks update-kubeconfig --name dybran-eks-tooling --region us-west-1 --kubeconfig kubeconfig`

![](./images/kbc.PNG)

Configure kubectl

After the cluster is created, you need to configure kubectl to connect to the cluster. Run the command

`$ aws eks --region us-east-1 update-kubeconfig --name deploy`

__DEPLOY APPLICATIONS WITH HELM__

__Helm__ serves as a robust package manager tailored for Kubernetes, the widely embraced open-source container orchestration framework. This tool streamlines the intricate process of deploying and administering applications within Kubernetes clusters. With Helm, you can succinctly define, install, and update even the most intricate Kubernetes applications using a collection of well-organized packages termed "charts."

In practice, I will leverage __Helm__ for deploying the manifest, as opposed to __kubectl__.

In the real-world scenario, __Helm__ stands out as the preeminent tool for deploying resources within Kubernetes, thanks to its extensive feature set, enabling the bundling of deployments into cohesive units.  Instead of juggling multiple YAML files independently, Helm ensures a more organized and efficient approach to managing Kubernetes deployments.

__Parameterising YAML manifests using Helm templates__

Using Helm templates to parameterize YAML manifests
Imagine that you have containerized the __Tooling app__ as an image named __"tooling-app"__ and want to deploy it with Kubernetes. Without Helm, you would have to manually create YAML manifests for the deployment, service and ingress. Then apply them to your Kubernetes cluster using `kubectl apply`. Initially, your application is at version 1, and the Docker image is labeled as "__tooling-app:1.0.0.__". A basic deployment manifest might look like this

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tooling-app-deployment
  labels:
    app: tooling-app
spec:
  replicas: 3
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: tooling-app
  template:
    metadata:
      labels:
        app: tooling-app
    spec:
      containers:
      - name: tooling-app
        image: "tooling-app:1.0.0"
        ports:
        - containerPort: 80
```

Now, let us consider the scenario where I need to deploy another version of the app, specifically __version 1.1.0__. If there are no changes required for the service or ingress, the deployment process can be simplified by creating a new version of the deployment manifest. In this new manifest, you would replace the image specified in the '__spec__' section with the image of __version 1.1.0__. Subsequently, you can reapply this modified manifest to the cluster, which will trigger an update of the deployment. This update follows a __rolling-update strategy__.

However, a notable drawback of this approach is that all the application-specific values, such as labels and image names are intermingled with the low-level, mechanical definition of the deployment manifest.

Helm addresses this issue by separating the configuration of a chart from its fundamental definition. To illustrate, instead of embedding the name of your app or the specific container image within the manifest, Helm allows you to provide these values during the installation of the chart into the cluster.

For instance, a simplified templated version of the previous deployment might resemble the following

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: "{{ template "name" . }}"
spec:
  replicas: 3
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: "{{ template "name" . }}"
  template:
    metadata:
      labels:
        app: "{{ template "name" . }}"
    spec:
      containers:
      - name: "{{ template "name" . }}"
        image: "{{ .Values.image.name }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
```

This illustration showcases several aspects of Helm templates:

- The template employs YAML as its foundation, utilizing the __{{ }}__ syntax to specify dynamic sections.

- Helm offers a range of variables that are populated during installation. For instance, __{{.Release.Name}}__ enables you to modify the resource's name dynamically by using the release name. When you install a Helm chart, it generates a release (a Helm concept, distinct from Kubernetes).

- You have the flexibility to create helper functions in external files. The __{{template "name"}}__ invocation yields a secure name for the application based on the Helm chart's name, which can be overridden if necessary. Utilizing these helper functions helps minimize redundancy in static values (such as "tooling-app") and reduces the likelihood of typographical errors.

- You can manually provide runtime configuration. The __{{.Values.image.name}}__ value, for instance, is derived from a predefined set of defaults or from values supplied during the execution of the `helm install` command. There are various methods to supply the required configuration values for chart installation via Helm. Typically, two approaches are employed:

   - A `values.yaml` file included in the chart itself. This file typically furnishes default configuration values and serves as documentation for the available configuration options.

   - When configuring via the command line, you can provide a configuration values file using the `-f` flag.

__setup Helm__

According to the [official documentation](https://helm.sh/docs/intro/install/), there are different options to installing Helm.

Download the binary

`$ wget https://get.helm.sh/helm-v3.13.1-linux-amd64.tar.gz`

![](./images/1.PNG)

Unarchive

`$ tar -zxvf helm-v3.13.1-linux-amd64.tar.gz`

Find the helm binary in the unpacked directory, and move it to its desired destination to make helm global

`$ sudo mv linux-amd64/helm /usr/local/bin/helm`

verify

`$ helm version`

![](./images/hh.PNG)

__DEPLOY JENKINS WITH HELM__

Utilizing publicly accessible charts to effortlessly employ the requisite tools.

Helm offers a remarkable capability: the ability to deploy pre-packaged applications directly from a public Helm repository with minimal configuration. A prime illustration of this capability is demonstrated when deploying Jenkins.

To find pre-packaged applications in the form of Helm Charts, visit the [Artifact Hub](https://artifacthub.io/packages/search).

Search for Jenkins.

Integrate the repository with Helm to simplify the download and deployment process.

`$ helm repo add jenkins https://charts.jenkins.io`

Update helm repo

`$ helm repo update`

![](./images/11.PNG)

Install the chart

`$ helm install jenkins jenkins/jenkins`












__PROBLEMS ENCOUNTERED__

- When I ran `terraaform apply`, I got this error

![](./images/error.PNG)

__UnsupportedAvailabilityZoneException:__ This exception is typically thrown when you attempt to perform an operation or allocate a resource in an availability zone that does not support that specific operation or resource type.

__SOLUTION__
I needed to change the AWS console region, which required me to reconfigure my 
__awscli__. To accomplish this, I executed a command to switch the region from __us-east-1__ to __us-west-1__.

`$ aws configure`

![](./images/21.PNG)

I created the bucket

`$ aws s3api create-bucket --bucket eks-terraform-deploy --create-bucket-configuration LocationConstraint=us-west-1`