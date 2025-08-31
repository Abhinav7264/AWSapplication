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
