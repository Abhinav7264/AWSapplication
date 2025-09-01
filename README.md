# Automated Deployment of Applications using Terraform and Ansible

> This project teaches how to build and deploy an application on an EC2 instance manually and using Ansible.
> The application is a simple website hosted on EC2 which holds employee details.
<img width="1055" height="419" alt="Screenshot 2025-08-31 at 2 14 56 PM" src="https://github.com/user-attachments/assets/691b8323-102e-4c05-bf6f-0a6f84638f5d" />

---

## Project Overview

This architecture is implemented in **3 parts**:

### Part 01 – Prerequisites and Preparation with Terraform
* Make a new directory called human-gov-infrastructure and clone my previous project for all the files - https://github.com/Abhinav7264/Awsinfrasetup.git
* move the content inside Awsinfrasetup folder to outside folder human-gov-infrastructure
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
terraform init
terraform plan         # Plan: resources to add
terraform apply
terraform destroy -auto-approve
```

---

### Step 02 – Create AWS IAM Role (Instance Profile)

This role provides **full access** to S3 and DynamoDB and is assigned to EC2 instances.

Add the following in `modules/aws_humangov_infrastructure/main.tf`:

```hcl
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

# Attach AmazonS3FullAccess policy to the IAM Role
resource "aws_iam_role_policy_attachment" "s3_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

# Attach AmazonDynamoDBFullAccess policy to the IAM Role
resource "aws_iam_role_policy_attachment" "dynamodb_full_access_role_policy_attachment" {
  role       = aws_iam_role.s3_dynamodb_full_access_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

# Create an IAM Instance Profile for the EC2 instance
resource "aws_iam_instance_profile" "s3_dynamodb_full_access_instance_profile" {
  name = "humangov-${var.state_name}-s3_dynamodb_full_access_instance_profile"
  role = aws_iam_role.s3_dynamodb_full_access_role.name

  tags = {
    Name = "humangov-${var.state_name}"
  }  
}
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




# 

## PART 2 - Deploying the application without using the power of Ansible

Manual deployment is often synonymous with repetitive, error-prone work. 

What's life like without automation?

- **High effort** - each step (build, upload, configuration, restart) depends on local scripts or hand-typed commands, requiring constant attention from the team.
- **Risk of human error** - a command in the wrong order or a forgotten config file can bring down the entire environment.
- **Inconsistency between servers** - reproducing the same state on all nodes is difficult; small differences accumulate "ghost" bugs.
- **Longer recovery time** - rollbacks and hotfixes become races against the clock because everything has to be redone manually.
- **Limited scalability** - adding new servers or micro-services means repeating the ritual, delaying releases and innovation.

"Without automation, the delivery cycle slows down, reliability drops and the team spends more time 'putting out fires' than creating value."

## Prerequisite:

- Deploying the Infrastructure with Terraform (Part 01)
- Generate an humangov-ec2-key.pem key-pair and have it stored in EC2 instance

---

## Step 01 - Downloading the 'HumanGov' Application Files

```bash
mkdir human-gov-application
cd human-gov-application

echo "*.zip" >> .gitignore
ls -a
cat .gitignore

mkdir src 
cd src
wget https://tcb-bootcamps.s3.amazonaws.com/tcb5001-devopscloud-bootcamp/v2/module4-ansible/humangov-app.zip
unzip humangov-app.zip
ls -l
```

## Step 02 - Committing the Changes to the "Local" Repository

```bash
cd ..
git status
git add .
git status
git commit -m "HumanGov app 1st commit"
git status
```

## Step 03 - Connecting to the Instance

```bash
ls -l /home/ec2-user/humangov-ec2-key.pem 
chmod 400 /home/ec2-user/humangov-ec2-key.pem 
ls -l /home/ec2-user/humangov-ec2-key.pem 

ssh -i /home/ec2-user/humangov-ec2-key.pem ubuntu@<humangov-california_PRIVATE_IP>
```

<aside>
💡

Before proceeding to execute the various commands below, think about what it would be like to automate these steps using Ansible.

</aside>

## Step 04 - Updating the Package List and Installing Updates

Update and upgrade 'apt' packages:

```bash
sudo apt update
sudo apt upgrade -y
```


## Step 05 - Installing the Necessary Packages

```bash
sudo apt-get install -y nginx python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools python3-venv unzip
```


## Step 06 - Allowing HTTP Access on the Firewall for Nginx

```bash
sudo ufw allow 'Nginx HTTP'
```

<aside>
💡

`ufw` = **Uncomplicated Firewall**

This is a simple firewall management tool for `iptables-based` Linux systems.

</aside>

## Step 07 - Creating a Project Directory Using Environment Variables

<aside>
💡

We're going to use the $project_path and $username variables in various parts of the project.

</aside>

```bash
# Listing to validate that no "humangov" exists
pwd
ls

# Creating and Listing Variables
export project_path=/home/ubuntu/humangov
echo $project_path
env | grep -i project_path

export username=ubuntu
echo $username
env | grep -i username

# Creating Project's Folder/Directory
mkdir -p $project_path
ls -l

# Adjusting Project's Folder/Directory Owner Permissions
sudo chown $username:$username $project_path
sudo chmod 0755 $project_path
```

<aside>
💡

Which Ansible module could we use to create Files, Directories and Set Permissions?

</aside>

## Step 08 - Creating a Python Virtual Environment

```bash
python3 -m venv $project_path/humangovenv

ls humangov/humangovenv/
```

## Step 09 - Copying the Application Files from EC2 (IDE)

```bash
exit
# Go to foder /home/ec2-user/human-gov-application/src
pwd
/home/ec2-user/human-gov-application/src

# Copying Application Files
scp -i /home/ec2-user/humangov-ec2-key.pem humangov-app.zip ubuntu@<PRIVATE_IP>:/home/ubuntu/humangov

# Validating Copy Process
ssh -i /home/ec2-user/humangov-ec2-key.pem ubuntu@<humangov-california_PRIVATE_IP>
ls humangov/
```

## Step 10 - Unzipping the Application Files

```bash
# Temporary Variable... do not exists, need to be setup again!
env | grep -i project_path
env | grep -i username

# Setting up the variables again
export project_path=/home/ubuntu/humangov
export username=ubuntu

# Creating a Variable for the "Project"
export project_name=humangov

# Listing Variables Values
env | grep -i project_path
env | grep -i username
env | grep -i project_name

# Unzinp application files
unzip $project_path/humangov-app.zip -d $project_path
ls humangov/
```

<aside>
💡

Variables defined with `export` in a terminal are **temporary** - they only exist during the current terminal session.

</aside>

---

### **How do you make these variables permanent? (Optional)**


To make these variables permanent, you need to add them to your user's environment configuration file. 

The most common location is:

```bash
~/.bashrc
```

### 🛠️ **Steps to make the variables permanent:**

**1 - Open the configuration file:**

```bash
nano ~/.bashrc

```

**2 - Add the lines at the end of the file:**

```bash
export project_path=/home/ubuntu/humangov
export username=ubuntu
```

**3 - Save and close the file (in nano: `Ctrl + O`, `Enter`, `Ctrl + X`).**

**4 - Reload the file to apply the changes:**

```bash
source ~/.bashrc
```

Every time the user starts a new terminal session, these variables will automatically be available.

---

## Step 11 - Installing 'Python' Packages via the 'requirements.txt' File in the Virtual Environment

```bash
cat $project_path/requirements.txt

$project_path/humangovenv/bin/pip install -r $project_path/requirements.txt
```

## Step 12 - Creating a 'systemd' Service for 'Gunicorn'

<aside>
💡

This step creates a**`systemd`** **service**  for **Gunicorn** (WSGI server) to automatically start and manage a Python web application as a system service - just like Nginx, for example.

</aside>

<aside>
💡

Pay attention to the fields highlighted in red that need changing!

***Validate with the resources created in AWS!***

</aside>

### Open a notepad, copy, paste and change the necessary fields!

### Afterwards, copy all the content and paste it into the terminal to run the `tee` command(<<EOL... EOL)

```bash
sudo tee /etc/systemd/system/$project_name.service <<EOL
[Unit]
Description=Gunicorn instance to serve $project_name
After=network.target

[Service]
User=$username
Group=www-data
WorkingDirectory=$project_path
Environment="PATH=$project_path/humangovenv/bin"
Environment="AWS_REGION=us-east-1"
Environment="AWS_DYNAMODB_TABLE=humangov-california-dynamodb"
Environment="AWS_BUCKET=humangov-california-s3-ku6m"
Environment="US_STATE=california"
ExecStart=$project_path/humangovenv/bin/gunicorn --workers 1 --bind unix:$project_path/$project_name.sock -m 007 $project_name:app

[Install]
WantedBy=multi-user.target
EOL
```

<aside>
💡

The file '**$project_name.sock**' will be created in later steps! This `.sock` is a **local communication file** used by Gunicorn to receive requests from Nginx with greater performance and security.

**`humangov.sock` is created automatically** by Gunicorn when it is started by `systemctl start humangov`.

</aside>

### The service has the following tasks:

- Defines where the project is`(WorkingDirectory`).
- Defines the user who will run the service`(User`).
- Defines environment variables (such as AWS region and tables).
- Sets up the Gunicorn start command`(ExecStart`).
- Allows the service to be managed with `systemctl` (start, stop, restart, etc.).

---

### Final result:

You will be able to run commands such as:

```bash
sudo systemctl start nome-do-servico
sudo systemctl enable nome-do-servico

```

And Gunicorn will be started automatically on boot and managed as a service.

### Before moving on to step 13, let's validate that the service has been properly created:

```bash
ls -l /etc/systemd/system/${project_name}.service
cat /etc/systemd/system/${project_name}.service
systemctl status ${project_name}.service
```

## Step 13 - Validate and Grant User Permission "Ubuntu"

```bash
ls -l /home/$username
echo $username
sudo chmod 0755 /home/$username
```

## Step 14 - Removing the 'nginx' Default Configuration File

<aside>
💡

**This command removes the default Nginx configuration, to avoid conflicts with our project.**

*It's like taking the "example site" offline to make room for HumanGov!*

</aside>

```bash
ls /etc/nginx/sites-enabled/default
cat /etc/nginx/sites-enabled/default

sudo rm /etc/nginx/sites-enabled/default
```

## Step 15 - Creating a 'proxy' configuration file for Nginx

In this step we are **creating a new configuration file for Nginx** with the name of our project.

- The Proxy should listen to port 80 (HTTP).
- Defining the server name.
- Directing requests to the Gunicorn socket (which is running the Python application).

This allows Nginx to act as an **"intermediary" between the browser and your Python application**.

```bash
sudo tee /etc/nginx/sites-available/$project_name <<EOL
server {
    listen 80;
    server_name $project_name www.$project_name;

    location / {
        include proxy_params;
        proxy_pass http://unix:$project_path/$project_name.sock;
    }
}
EOL
```

### Listing "available" and "enabled" sites

```bash
# Listing Sites Available
ls /etc/nginx/sites-available/

# Listing Sites Enabled
ls /etc/nginx/sites-enabled/
```

## Step 16 - Enabling and starting 'Gunicorn' services

<aside>
💡

- `enable`: makes the service start automatically when the system turns on.
- `start`: starts the service now.
- `status`: shows whether the service is running, with possible error or success messages.
</aside>

```bash
sudo systemctl enable $project_name

sudo systemctl start $project_name

sudo systemctl status $project_name

# Validating "humangov.sock" file creation
ls humangov/
```

## Step 17 - Enabling the 'Nginx' Configuration File

<aside>
💡

This command activates your site's configuration in Nginx, allowing it to recognize and use that configuration on startup.

</aside>

```bash
# Listing Sites Enabled
ls /etc/nginx/sites-enabled/

# Enabling
sudo ln -s /etc/nginx/sites-available/$project_name /etc/nginx/sites-enabled/

# Listing Sites Enabled
ls /etc/nginx/sites-enabled/
```

## Step 18 - Restarting the 'Nginx' service and the 'humangov service':

```bash
sudo systemctl restart $project_name

sudo systemctl status $project_name

sudo systemctl restart nginx

sudo systemctl status nginx
```

## Step 19 - Testing the application!

- EC2_humangov-california_Public**DNS**
    - Add employee

([Driving License - Sample.pdf](https://github.com/user-attachments/files/22069972/Driving.License.-.Sample.pdf)
)



<aside>
💡

### **Imagine doing all these steps for more than 50 states?**

</aside>

## Step 20 - Validating Resources on AWS (DynamoDB and S3)

- **DynamoDB**
    - **humangov-california-dynamodb**
        - Explore items
- **S3**
    - **humangov-california-s3-xxxx**
        - Objects

<aside>
💡

## This app has a bug! 🪲

### When deleting an employee, the data in the DynamoDB table is deleted, but the file in the bucket remains!

### It's an intentional bug that we'll fix in the Python module!

</aside>

## For now, let's delete manually (just the object)! 🔥

## Step 21 - Destroying Resources

```bash
exit
cd ~/human-gov-infrastructure/terraform
terraform destroy -auto-approve
```

🔒 ***Close Remote Connection***

**⛔ Stop Your IDE/EC2 Instance!**


