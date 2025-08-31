# Hands-on Project: Implementation - Part 1

![Architecture Diagram](attachment:6a6c145f-3f15-49a1-a38e-805f44e71391:image.png)

This architecture will be implemented in **3 parts**:

---

### Part 01 - **Prerequisites and Preparation with Terraform**
In this step, we prepare the infrastructure provisioning files using **Terraform**.  
This includes creating **IAM Roles**, **Security Groups**, and other essential resources for the application.  
The goal is to ensure that all components are ready before deployment.

---

### Part 02 - **Manual Deployment: Understanding the Effort and Risks**
Here we deploy the application manually, **without automation**.  
This highlights the **effort involved** and the **risks of human error** when automation is not used.

---

### Part 03 - **Automated Deployment with Ansible**
Finally, we automate deployment with **Ansible** using **roles** and **playbooks**.  
This makes the process **repeatable, reliable, and efficient**, while minimizing human error.

---

# Part 01 - Prerequisites and Preparation with Terraform

---

## Prerequisites (Cloud9 Only)
- Create an **IAM user** with `AdministratorAccess` permissions.  
- Disable Cloud9 temporary credentials:  
  `Settings > AWS Settings > Credentials > Turn Off AWS managed temporary credentials`.  
- Configure credentials for the IAM user:  
  ```bash
  aws configure
Step 01 - Validate Current Terraform Code
bash
Copy code
cd human-gov-infrastructure/terraform
terraform plan         # Plan: 5 to add
terraform apply
terraform destroy -auto-approve
Step 02 - Create AWS IAM Role (Instance Profile)
Define an IAM Role with full access to S3 and DynamoDB, then create an Instance Profile for EC2.

Add the following to:
modules/aws_humangov_infrastructure/main.tf

hcl
Copy code
# IAM Role for EC2 to access S3 and DynamoDB
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

# Attach Policies
resource "aws_iam_role_policy_attachment" "s3_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role_policy_attachment" "dynamodb_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

# IAM Instance Profile
resource "aws_iam_instance_profile" "s3_dynamodb_full_access_instance_profile" {
  name = "humangov-${var.state_name}-s3_dynamodb_full_access_instance_profile"
  role = aws_iam_role.s3_dynamodb_full_access_role.name

  tags = {
    Name = "humangov-${var.state_name}"
  }  
}
Step 03 - Associate Instance Profile with EC2
Inside the aws_instance block, add:

hcl
Copy code
iam_instance_profile = aws_iam_instance_profile.s3_dynamodb_full_access_instance_profile.name
Step 04 - Add Application Access Rules in Security Group
Replace existing ingress rules with:

hcl
Copy code
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
💡 Alternative: Replace the entire main.tf with the complete adjusted code (provided in the project documentation).

Step 05 - Provision Infrastructure with Terraform
bash
Copy code
terraform plan    # Plan: 9 to add
terraform apply -auto-approve
Step 06 - Validate on AWS
Confirm EC2 is created.

Ensure the IAM Role is attached.



Step 07 - Commit Changes to Git
bash
Copy code
git status
git add .
git commit -m "Added IAM Role and SG updates to Terraform"
Step 08 - Destroy Resources
bash
Copy code
terraform destroy -auto-approve
🔒 Close Remote Connection
⛔ Stop your IDE/EC2 instance!
