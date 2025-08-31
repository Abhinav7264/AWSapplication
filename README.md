# Automated Deployment of Applications using Terraform and Ansible

> This project teaches how to build and deploy an application on an EC2 instance manually and using Ansible.
> The application is a simple website hosted on EC2 which holds employee details.
<img width="1055" height="419" alt="Screenshot 2025-08-31 at 2 14 56 PM" src="https://github.com/user-attachments/assets/691b8323-102e-4c05-bf6f-0a6f84638f5d" />

---

## Project Overview

This architecture is implemented in **3 parts**:

### Part 01 – Prerequisites and Preparation with Terraform
* clone my previous project for all the files - https://github.com/Abhinav7264/Awsinfrasetup.git
* Adjust infrastructure provisioning files using **Terraform**.
* Create **Roles**, **Security Groups**, and other essential resources.
* Ensure all components are prepared for deployment.

### Part 02 – Manual Deployment: Understanding Effort and Risks

* Deploy the application manually without automation.
* Highlight effort and possible sources of error.
* Demonstrates the importance of automation.

### Part 03 – Automated Deployment with Ansible

* Use Roles and Playbooks for standardized deployment.
* Drastically reduces execution time and minimizes human error.

---

## Part 01 – Prerequisites and Preparation with Terraform

### Step 01 – Validate Current Terraform Code - optional

Optional: terraform code contains main.tf which builds just EC2 instances.

```bash
cd human-gov-infrastructure/terraform
terraform plan         # Plan: resources to add
terraform apply
terraform destroy -auto-approve
```

---

### Step 02 – Create AWS IAM Role (Instance Profile)

This role provides **full access** to S3 and DynamoDB and is assigned to EC2 instances.

Add the following in `modules/aws_humangov_infrastructure/main.tf`:

```hcl
resource "aws_iam_role" "s3_dynamodb_full_access_role" {...}
resource "aws_iam_role_policy_attachment" "s3_full_access_role_policy_attachment" {...}
resource "aws_iam_role_policy_attachment" "dynamodb_full_access_role_policy_attachment" {...}
resource "aws_iam_instance_profile" "s3_dynamodb_full_access_instance_profile" {...}
```

---

### Step 03 – Associate Instance Profile in EC2

Inside the EC2 resource block `aws_instance "state_ec2"`:

```hcl
iam_instance_profile = aws_iam_instance_profile.s3_dynamodb_full_access_instance_profile.name
```

---

### Step 04 – Add Application Access Rules in Security Group

Replace existing ingress rules with:

```hcl
ingress { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
ingress { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
ingress { from_port = 5000, to_port = 5000, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
ingress { from_port = 0, to_port = 0, protocol = "-1", security_groups = ["<YOUR_CLOUD9_SECGROUP_ID>"] }
egress { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
```

---

### Step 04a – Full `main.tf` File

```hcl
resource "aws_security_group" "state_ec2_sg" {
    name = "humangov-${var.state_name}-ec2-sg"
    description = "Allow traffic on ports 22 and 80"
    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 5000
        to_port     = 5000
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        security_groups = ["<YOUR_CLOUD9_SECGROUP_ID"]
    }
    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
    tags = {
        Name = "humangov-${var.state_name}"
    }
}

resource "aws_instance" "state_ec2" {
    ami = "ami-007855ac798b5175e"
    instance_type = "t2.micro"
    key_name = "humangov-ec2-key"
    vpc_security_group_ids = [aws_security_group.state_ec2_sg.id]
    iam_instance_profile = aws_iam_instance_profile.s3_dynamodb_full_access_instance_profile.name
    tags = {
        Name = "humangov-${var.state_name}"
    }
}

resource "aws_dynamodb_table" "state_dynamodb" {
    name = "humangov-${var.state_name}-dynamodb"
    billing_mode = "PAY_PER_REQUEST"
    hash_key = "id"
    attribute {
        name = "id"
        type = "S"
    }
    tags = {
        Name = "humangov-${var.state_name}"
    }
}

resource "random_string" "bucket_suffix" {
    length = 4
    special = false
    upper = false
}

resource "aws_s3_bucket" "state_s3" {
    bucket = "humangov-${var.state_name}-s3-${random_string.bucket_suffix.result}"
    tags = {
        Name = "humangov-${var.state_name}"
    }
}

resource "aws_iam_role" "s3_dynamodb_full_access_role" {
  name = "humangov-${var.state_name}-s3_dynamodb_full_access_role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
  tags = { Name = "humangov-${var.state_name}" }
}

resource "aws_iam_role_policy_attachment" "s3_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role_policy_attachment" "dynamodb_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

resource "aws_iam_instance_profile" "s3_dynamodb_full_access_instance_profile" {
  name = "humangov-${var.state_name}-s3_dynamodb_full_access_instance_profile"
  role = aws_iam_role.s3_dynamodb_full_access_role.name
  tags = { Name = "humangov-${var.state_name}" }
}
```

---

### Step 05 – Provision Infrastructure on AWS

```bash
terraform plan
terraform apply -auto-approve
```

---

### Step 06 – Validate EC2 and IAM Role

Check AWS console to confirm EC2 instance is running and IAM Role is attached
