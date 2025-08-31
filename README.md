Perfect! I’ve converted all your detailed instructions into a polished README.md ready for your Git repository. You can copy and paste it directly.

# HumanGov Infrastructure Project

> Hands-on project to implement HumanGov infrastructure using Terraform and Ansible.

---

## Project Overview

This architecture is implemented in **3 parts**:

### Part 01 – Prerequisites and Preparation with Terraform
- Adjust infrastructure provisioning files using **Terraform**.
- Create **Roles**, **Security Groups**, and other essential resources.
- Ensure all components are prepared for deployment.

### Part 02 – Manual Deployment: Understanding Effort and Risks
- Deploy the application manually without automation.
- Highlight effort and possible sources of error.
- Demonstrates the importance of automation.

### Part 03 – Automated Deployment with Ansible
- Use Roles and Playbooks for standardized deployment.
- Drastically reduces execution time and minimizes human error.

---

## Part 01 – Prerequisites and Preparation with Terraform

### Prerequisites (Cloud9)

1. Create an IAM user with **Admin permissions**.
2. Disable **Cloud9 temporary credentials**:
   - Settings > AWS Settings > Credentials > Turn off "AWS managed temporary credentials".
3. Configure credentials for the new IAM user:

```bash
aws configure

Step 01 – Validate Current Terraform Code
cd human-gov-infrastructure/terraform
terraform plan         # Plan: resources to add
terraform apply
terraform destroy -auto-approve

Step 02 – Create AWS IAM Role (Instance Profile)

This role provides full access to S3 and DynamoDB and is assigned to EC2 instances.

Add the following in modules/aws_humangov_infrastructure/main.tf:

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

  tags = {
    Name = "humangov-${var.state_name}"
  }  
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

  tags = {
    Name = "humangov-${var.state_name}"
  }  
}

Step 03 – Associate Instance Profile in EC2

Inside the EC2 resource block aws_instance "state_ec2":

iam_instance_profile = aws_iam_instance_profile.s3_dynamodb_full_access_instance_profile.name

Step 04 – Add Application Access Rules in Security Group

Replace existing ingress rules with:

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
  from_port       = 0
  to_port         = 0
  protocol        = "-1"
  security_groups = ["<INSERT_YOUR_EC2/IDE_SECURITY_GROUP_ID>"]
}

egress {
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}

Step 05 – Provision Infrastructure on AWS
terraform plan    # Plan resources to add
terraform apply -auto-approve

Step 06 – Validate EC2 and IAM Role

Check AWS console to confirm EC2 instance is running and IAM Role is attached.

Step 07 – Commit Changes to Local Repository
git status
git add .
git commit -m "Added IAM Role to Terraform module aws_humangov_infrastructure/main.tf"

Step 08 – Destroy Resources
terraform destroy -auto-approve


🔒 Close Remote Connection and stop IDE/EC2 instance.

Notes

Use humangov-ec2-key SSH key for EC2 instances.

Adjust <YOUR_CLOUD9_SECGROUP_ID> for your environment.

Ensure all AWS credentials are properly configured.

Directory Structure
.
├── modules
│   └── aws_humangov_infrastructure
├── terraform
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
