---
title: "AWS Bottlerocket - Significantly reducing container startup time"
date: "2024-12-13"
categories: 
  - "cloud"
  - "security"
tags: 
  - "aws"
  - "eks"
  - "containers"
  - "bottlerocket"
  - "terraform"
  - "opentofu"
coverImage: "bottlerocket.jpg"
---

I recently stumbled across [this article](https://aws.amazon.com/blogs/containers/reduce-container-startup-time-on-amazon-eks-with-bottlerocket-data-volume/) from October 2023, which demonstrates an idea of using [AWS's Bottlerocket](https://aws.amazon.com/bottlerocket/) in your EKS cluster with images already pulled to an EBS snapshot, which is added to the node. This concept essentially removes any container image pulling overhead, and is a simple alternative to things like [spegel](https://github.com/spegel-org/spegel).

That article is a little outdated, and forces users to use cloudformation which I loathe.

This post will take you through an alternative and up-to-date (and more manual) method of configuring Bottlerocket and using it in your EKS cluster with [karpenter](https://karpenter.sh/).

A video demonstration of the result is available [here](https://youtu.be/YB-6GOeR34g).

# Concept
You can read the linked article and read more about [Bottlerocket](https://aws.amazon.com/bottlerocket/), but at a high level we essentially provision a Bottlerocket EC2 instance, pull the images we want and snapshot the disk. Then, in your Karpenter configuration, specify the snapshot ID. When the cluster is scaled, Karpenter will launch a new node, register and initialize it (~30 seconds), and deploy your containers. As the container images already exist (in the mounted volume), they will start up instantly. It essentially means that in a scaling event the only overhead is the node registration and initialization ⚡

This concept is really useful if you have to pull the same couple of large containers regularly in an auto-scaled EKS cluster. 

# Getting started
There is also an [AWS labs page](https://awslabs.github.io/data-on-eks/docs/bestpractices/preload-container-images) on the concept, which links to the [aws-samples bottlerocket-images-cache Github repo](https://github.com/aws-samples/bottlerocket-images-cache/tree/main).

However, that requires the use of `eksctl` and cloudformation, which I find grim.

Instead, here is some terraform which will deploy a Bottlerocket EC2 instance, with access to the machine via Systems Manager (look how simple it is ❤).
```
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.80.0"
    }
  }
}

provider "aws" { }

locals {
  # newer versions dont seem to start the SSM agent properly...
  eks_version = "1.22"
}

# aws ec2 describe-images --filters "Name=name,Values=bottlerocket-aws-k8s-1.31-x86_64*" 
data "aws_ami" "bottlerocket" {
  most_recent = true

  filter {
    name   = "name"
    values = ["bottlerocket-aws-k8s-${local.eks_version}-x86_64*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["amazon"]
}
resource "aws_iam_role" "bottlerocket" {
    name = "bottlerocket"
    assume_role_policy = jsonencode(
        {
            Version = "2012-10-17"
            Statement = [
                {
                    Action = "sts:AssumeRole"
                    Effect = "Allow"
                    Sid = "EC2SSMIP"
                    Principal = {
                        Service = "ec2.amazonaws.com"
                    }
                }
            ]
        }
    )
}

resource "aws_iam_role_policy_attachment" "bottlerocket" {
    role = aws_iam_role.bottlerocket.name
    policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# requires ECR perms for startup. 
# https://github.com/bottlerocket-os/bottlerocket/issues/1139#issue-713339180
resource "aws_iam_role_policy_attachment" "bottlerocket2" {
    role = aws_iam_role.bottlerocket.name
    policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_instance_profile" "ec2instanceprofile" {
    name = "BottleRocketIP"
    role = aws_iam_role.bottlerocket.name
}

resource "aws_instance" "bottlerocket" {
  ami           = data.aws_ami.bottlerocket.id
  instance_type = "m5.large"

  iam_instance_profile = aws_iam_instance_profile.ec2instanceprofile.name

  metadata_options {
    http_tokens = "required" # Enforces IMDSv2
  }

  user_data = <<-EOT
  # The admin host container provides SSH access and runs with "superpowers".
  # It is disabled by default, but can be disabled explicitly.
  [settings.host-containers.admin]
  enabled = true

  # The control host container provides out-of-band access via SSM.
  # It is enabled by default, and can be disabled if you do not expect to use SSM.
  # This could leave you with no way to access the API and change settings on an existing node!
  [settings.host-containers.control]
  enabled = true
EOT
  tags = {
    Name = "bottlerocket"
  }
}


output "bottlerocket_ami_id" {
  value = data.aws_ami.bottlerocket.id
}

output "ec2_id" {
  value = aws_instance.bottlerocket.id
}
```

I found out the hard way, but the `AmazonEC2ContainerRegistryReadOnly` role **is required** for the Bottlerocket OS to start up successfully.
I also had some issues with the newer versions of Bottlerocket, which wouldnt start the AWS SSM agent. I ended up using an older bottlerocket version in the EC2 for the snapshotting, but in my EKS cluster, i use the latest version.
Bottlerocket is intentionally locked down which makes it difficult to configure - you cant even (I couldnt) access properly via the serial console!

The above will provision a **public** facing Bottlerocket EC2 instance - but I wouldnt worry about any attackers getting into it, its hard enough when you're the one who provisioned it! As we need to use SSM to open a session, the above terraform assumes that your default VPC configuration allows all traffic in and out from the public internet.
It dosent really matter that this instance is public facing, as its just temporary anyway. We want to just pull our images, and snapshot the disk.

Once the instance is running, open up a SSM session. If you've got the [AWS Session Manager plugin for the AWS CLI](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) installed, and are running OpenTofu, then you can just use this:
```
aws ssm start-session --target $(tofu output -raw ec2_id)
```

Bottlerocket has two containers, the admin and host container. When pulling container images via the `apiclient`, we need to make sure to not pull via the hosts containerd instance.

Deconstructing the [official snapshot.sh](https://github.com/aws-samples/bottlerocket-images-cache/blob/main/snapshot.sh#L187) script, gives us just three remaining steps to complete to build our snapshot.
1. Stop the kubelet service
```
apiclient exec admin sheltie systemctl stop kubelet
```

2. Pull images
For the exact version of bottlerocket I specified here, you can use this command:
```
apiclient exec admin sheltie ctr -a /run/dockershim.sock -n k8s.io images pull --label io.cri-containerd.image=managed docker.io/splunk/splunk:latest
```

However, if you manage to get an SSM session to a newer bottlerocket OS, I believe the argument to specify the containerd runtime has changed.
```
apiclient exec admin sheltie ctr -a /run/containerd/containerd.sock -n k8s.io images pull --label io.cri-containerd.image=managed docker.io/splunk/splunk:latest
```

For your container images, you'll have to specify the full url. For docker hub, its typically either `registry.hub.docker.com` or `docker.io`. In my experience, it just depends on the images, but either one of those normally works.

You can of course pull from ECR also, but i've not tried that. The `snapshot.sh` script [specifies a ECR password as an argument](https://github.com/aws-samples/bottlerocket-images-cache/blob/main/snapshot.sh#L214) to the `image pull` command.


3. Once you've pulled your images, just exit out of the SSM session, and snapshot the disk with the following:
a. Find the volume ID of the `bottlerocket` instance
```
aws ec2 describe-instances --filters "Name=tag:Name,Values=bottlerocket" --query "Reservations[0].Instances[0].BlockDeviceMappings[?DeviceName=='/dev/xvdb'].Ebs.VolumeId" --no-cli-pager --output text
```

b. Create the snapshot
```
aws ec2 create-snapshot --volume-id vol-07ea39c26d269afff --description "Bottlerocket Data Volume snapshot" --query "SnapshotId" --output text
```

Note the outputted snapshot ID, as you'll need to specify that in your Karpenter configuration.

You can then run `tofu destroy` to destroy the infrastructure. Your snapshot will remain, as its not part of the terraform. Snapshots cost like `5p` per GB a month, so its a not a big deal leaving it hanging around.

# Karpenter
In your Karpenter configuration, is as simple as specifying the `snapshotId` in your `EC2NodeClass` definition.
```
resource "kubectl_manifest" "ec2_node_class" {
  depends_on = [ helm_release.karpenter ]
  yaml_body  = <<YAML
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  blockDeviceMappings:
    - deviceName: /dev/xvdb
      ebs:
        volumeSize: 150Gi
        volumeType: gp3
        snapshotID: snap-02d793dc3f51bc898
  role: ${aws_iam_role.node-group.name}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${aws_eks_cluster.labs.name}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${aws_eks_cluster.labs.name}
YAML
}
```
Notice that the amiFamily is Bottlerocket, which will automatically use the mounted `/dev/xvdb` for local container images.

# Demo
[Here](https://youtu.be/YB-6GOeR34g) is a video demonstration of me deploying a large container (`splunk/splunk:latest` @ `3.28GB`) , which causes a scaling event, which uses this fancy deployment to remove the image pull time overhead entirely. It means that essentially my container is running ~30 seconds after causing the scaling event.
The only remaining overhead, is Karpeneter automatically provisioning a new EC2 instance (node), and registering and initialising the node ⚡
