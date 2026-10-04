# Secure AWS Infrastructure with Terraform

A security-focused Infrastructure as Code (IaC) project that provisions, hardens, audits, and validates an AWS environment using Terraform, AWS Systems Manager, IAM, and Prowler.

The goal of this project is to demonstrate how cloud infrastructure can be created entirely as code, secured using AWS security controls, audited with an independent cloud security tool, remediated through Terraform, and verified with before/after security scans.

## Project Objectives

- Provision AWS infrastructure entirely with Terraform
- Create a custom VPC and subnet
- Configure Internet routing
- Deploy an Ubuntu EC2 instance
- Encrypt EC2 root storage
- Enforce IMDSv2
- Implement an EC2 IAM role
- Apply least-privilege permissions for instance management
- Replace public SSH administration with AWS Systems Manager Session Manager
- Remove unnecessary inbound access
- Audit the AWS environment with Prowler
- Analyze security findings instead of blindly applying scanner recommendations
- Remediate relevant findings through Terraform
- Verify remediation with a second security scan
- Demonstrate an insecure SSH configuration without applying it to AWS

## Architecture

```text
AWS
└── VPC 10.0.0.0/16
    │
    ├── Public Subnet 10.0.1.0/24
    │
    ├── Internet Gateway
    │
    ├── Public Route Table
    │   └── 0.0.0.0/0 → Internet Gateway
    │
    └── EC2 Instance
        ├── Ubuntu 24.04 LTS
        ├── t3.micro
        ├── Encrypted EBS Root Volume
        ├── IMDSv2 Required
        ├── CloudWatch Detailed Monitoring
        ├── Security Group
        │   ├── No Inbound Rules
        │   └── Outbound Traffic Allowed
        └── IAM Instance Profile
            └── AmazonSSMManagedInstanceCore
```

Administrative access is performed through AWS Systems Manager Session Manager instead of an Internet-facing SSH service.

## Technologies Used

- AWS
- Terraform
- Amazon VPC
- Amazon EC2
- AWS IAM
- AWS Systems Manager Session Manager
- Amazon CloudWatch
- Amazon EBS
- Prowler
- AWS CLI
- Ubuntu Linux
- Git
- GitHub

## Infrastructure as Code

Terraform is used as the source of truth for the AWS infrastructure.

The standard workflow used throughout the project is:

```text
Terraform Code
      ↓
terraform fmt
      ↓
terraform validate
      ↓
terraform plan
      ↓
Review Planned Changes
      ↓
terraform apply
      ↓
AWS Infrastructure
```

The final infrastructure state was validated with:

```text
No changes. Your infrastructure matches the configuration.
```

This confirms that the Terraform configuration, Terraform state, and deployed AWS infrastructure are synchronized.

## Terraform Resources

The project manages the following AWS resources:

- VPC
- Public Subnet
- Internet Gateway
- Route Table
- Route Table Association
- Security Group
- Security Group Egress Rule
- EC2 Instance
- EC2 Key Pair
- IAM Role
- IAM Instance Profile
- IAM Managed Policy Attachment
- Ubuntu AMI Data Source

## Network Configuration

The project uses a dedicated VPC:

```text
10.0.0.0/16
```

A subnet is created inside the VPC:

```text
10.0.1.0/24
```

The subnet uses a route table containing:

```text
0.0.0.0/0 → Internet Gateway
```

This provides an Internet route for resources placed in the subnet.

## Dynamic Ubuntu AMI Selection

Instead of hardcoding an AMI ID, Terraform queries the latest matching official Canonical Ubuntu 24.04 LTS image.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  owners = ["099720109477"]
}
```

The EC2 instance references the selected AMI dynamically:

```hcl
ami = data.aws_ami.ubuntu.id
```

This avoids manually hardcoding a region-specific AMI identifier.

## EC2 Security Hardening

The EC2 instance includes several infrastructure-level security controls.

### Encrypted Root Volume

The root EBS volume is encrypted:

```hcl
root_block_device {
  encrypted = true
}
```

This protects data stored on the root volume at rest.

### IMDSv2 Enforcement

Instance Metadata Service Version 2 is required:

```hcl
metadata_options {
  http_tokens = "required"
}
```

This prevents tokenless IMDSv1 access and reduces the risk of unauthorized access to instance metadata and temporary IAM credentials.

Prowler verification:

```text
ec2_instance_imdsv2_enabled
PASS
```

### CloudWatch Detailed Monitoring

Prowler initially detected that EC2 Detailed Monitoring was disabled.

Initial result:

```text
ec2_instance_detailed_monitoring_enabled
FAIL
```

Terraform remediation:

```hcl
monitoring = true
```

The change was applied as an in-place update:

```text
Plan: 0 to add, 1 to change, 0 to destroy.
```

After remediation, Prowler was executed again.

Final result for the project EC2 instance:

```text
ec2_instance_detailed_monitoring_enabled
PASS
```

This provides a real security audit and remediation cycle:

```text
Prowler Audit
    ↓
Finding Detected
    ↓
Terraform Remediation
    ↓
terraform plan
    ↓
terraform apply
    ↓
Prowler Re-scan
    ↓
PASS
```

## IAM and Least-Privilege Design

The EC2 instance uses a dedicated IAM role:

```text
devsecops-ec2-ssm-role
```

The role is attached to the EC2 instance through an IAM Instance Profile.

The only managed policy attached to the role is:

```text
AmazonSSMManagedInstanceCore
```

Verification showed:

```text
Attached Managed Policies:
- AmazonSSMManagedInstanceCore

Inline Policies:
- None
```

The EC2 role does not include broad permissions such as:

```text
AdministratorAccess
AmazonEC2FullAccess
AmazonS3FullAccess
Custom wildcard administrative permissions
```

The IAM role exists specifically to support Systems Manager functionality.

## Secure Administration with AWS Systems Manager

The instance was initially accessed through SSH using a dedicated SSH key and a Security Group rule restricted to a single administrator IPv4 address using `/32`.

Initial management model:

```text
Administrator
    ↓
Public IP
    ↓
TCP/22
    ↓
EC2
```

Because the administrator Internet connection uses a changing public IP address, this approach also created an operational dependency on continuously updating the Security Group.

AWS Systems Manager Session Manager was therefore configured and tested.

The EC2 instance successfully registered with Systems Manager:

```text
PingStatus: Online
Platform: Ubuntu
```

A Session Manager connection was then established successfully:

```text
AWS CLI
    ↓
AWS Systems Manager
    ↓
SSM Agent
    ↓
EC2
```

The session was verified on the instance:

```text
whoami
ssm-user
```

Administrative privileges were also verified through:

```text
sudo whoami
root
```

After Systems Manager access was confirmed, the SSH ingress rule was removed completely.

Final Security Group state:

```text
Inbound:
No rules

Outbound:
Allowed
```

An SSH connection attempt after the rule was removed resulted in a timeout, while Systems Manager access continued to work successfully.

This confirms that EC2 administration no longer requires:

- Public SSH access
- TCP port 22
- A static administrator public IP address
- Direct SSH key-based Internet administration

## Insecure SSH Configuration Demonstration

As part of the security exercise, an intentionally insecure SSH rule was temporarily added to the Terraform configuration:

```hcl
resource "aws_vpc_security_group_ingress_rule" "insecure_ssh_demo" {
  security_group_id = aws_security_group.web.id

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 22
  to_port     = 22
  ip_protocol = "tcp"

  description = "INSECURE DEMO - SSH open to the Internet"
}
```

Terraform showed:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

The planned rule would have exposed SSH to:

```text
0.0.0.0/0
```

which represents the entire IPv4 Internet.

The insecure configuration was never applied to AWS.

It was removed from the Terraform configuration, followed by another validation and plan.

Final result:

```text
No changes. Your infrastructure matches the configuration.
```

This demonstrates the ability to identify a dangerous infrastructure change during the Terraform planning stage before it reaches production infrastructure.

## Cloud Security Audit with Prowler

Prowler 5.44.0 was used to audit the deployed AWS environment.

An initial targeted IMDSv2 scan verified that the project EC2 instance correctly required IMDSv2.

Result:

```text
PASS
```

A broader EC2 service scan was then performed.

The account-level EC2 audit produced both PASS and FAIL findings.

Project-specific findings included:

| Finding | Severity | Initial Status | Final Decision |
|---|---|---|---|
| IMDSv2 enforcement | High if failed | PASS | Secure configuration retained |
| EC2 Detailed Monitoring | Low | FAIL | Remediated through Terraform → PASS |
| EC2 has a public IP | Medium | FAIL | Accepted lab architecture limitation |
| Public IP with IAM Instance Profile | High | FAIL | Accepted lab architecture limitation |

## Prowler Remediation Example

The most direct remediation performed during the audit was Detailed Monitoring.

Before:

```text
EC2 Detailed Monitoring
FAIL
```

Terraform remediation:

```hcl
monitoring = true
```

After:

```text
EC2 Detailed Monitoring
PASS
```

This demonstrates that scanner findings were not only identified but also traced back to Infrastructure as Code, remediated through Terraform, applied safely, and independently re-verified.

## Accepted Lab Limitations

Some Prowler findings were intentionally not remediated because they would require expanding the project architecture beyond its current scope.

### Public EC2 IP Address

The EC2 instance currently receives a public IPv4 address.

Prowler reports this as a security finding because a production workload should generally avoid direct public addressing when it is not required.

The current lab mitigates this risk by using:

```text
No inbound Security Group rules
IMDSv2 enforcement
SSM-based administration
Restricted EC2 IAM permissions
```

A more production-oriented architecture would use:

```text
Private Subnet
    ↓
EC2 Without Public IP
    ↓
NAT Gateway and/or VPC Endpoints
    ↓
AWS Services
```

Administrative access would continue through Systems Manager.

This improvement is intentionally documented rather than added to the current lab to keep the project focused on its original learning objectives.

### Terraform Operator Permissions

The lab Terraform operator account currently has administrative permissions for infrastructure provisioning.

This simplifies the learning environment but is broader than a production Terraform execution identity should have.

A production environment should use a dedicated, scoped Terraform execution role with only the permissions required to manage the intended infrastructure.

### IAM Trust Policy Audit Finding

Prowler identified a cross-service confused deputy prevention finding for the EC2 service role trust policy.

The role uses the standard EC2 service trust relationship:

```text
Principal: ec2.amazonaws.com
Action: sts:AssumeRole
```

The finding was reviewed in the context of an EC2 instance-profile role and was documented rather than modified solely to make the scanner result green.

This reflects an important security engineering principle:

```text
Scanner Finding
    ↓
Understand Context
    ↓
Validate Applicability
    ↓
Remediate or Document
```

Security scanners are decision-support tools and should not be followed blindly.

## Security Audit Workflow

The project demonstrates the following DevSecOps workflow:

```text
DESIGN
  ↓
DEFINE INFRASTRUCTURE AS CODE
  ↓
VALIDATE
  ↓
PLAN
  ↓
DEPLOY
  ↓
HARDEN
  ↓
AUDIT
  ↓
ANALYZE FINDINGS
  ↓
REMEDIATE THROUGH CODE
  ↓
RE-SCAN
  ↓
VERIFY
```

## Sensitive File Handling

Local Terraform and security-audit artifacts are excluded from Git.

The `.gitignore` includes:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
.venv/
output/
crash.log
```

This prevents local Terraform state, environment-specific values, Python virtual environments, and raw Prowler reports from being committed to the repository.

The Terraform provider lock file is intentionally committed to improve dependency reproducibility.

## Repository Structure

```text
iac-secure-cloud-infra/
├── .gitignore
├── .terraform.lock.hcl
├── README.md
├── provider.tf
├── main.tf
└── iam.tf
```

Local-only files and directories such as `.terraform/`, `.venv/`, Terraform state files, and Prowler output are excluded from version control.

## Key Security Lessons

This project demonstrates several important cloud and DevSecOps principles:

1. Infrastructure should be defined and reviewed as code.
2. `terraform plan` should be reviewed before infrastructure changes are applied.
3. Public administration ports should not remain open when safer alternatives exist.
4. AWS Systems Manager can remove the requirement for Internet-facing SSH administration.
5. EC2 workloads should use IAM roles instead of static AWS access keys.
6. IAM permissions should be limited to the workload's actual requirements.
7. EC2 metadata should require IMDSv2.
8. Storage encryption should be enabled.
9. Security scanners should validate deployed infrastructure independently from Terraform.
10. Scanner findings must be analyzed in architectural context.
11. Relevant findings should be remediated through Infrastructure as Code.
12. Remediation should be verified with a second scan.
13. A security fix should not blindly break required system functionality.
14. Lab limitations should be documented instead of hidden.
15. The secure final state should remain reproducible through Terraform.

## Final State

The final project state includes:

```text
Terraform-managed AWS infrastructure     ✅
Custom VPC                               ✅
Custom subnet                            ✅
Internet routing                         ✅
EC2 provisioned through Terraform        ✅
Encrypted EBS root volume                ✅
IMDSv2 required                          ✅
CloudWatch Detailed Monitoring           ✅
Dedicated EC2 IAM role                   ✅
SSM-based administration                 ✅
No inbound Security Group rules          ✅
Public SSH removed                       ✅
Prowler security audit                   ✅
Security finding remediation             ✅
Before/after verification                ✅
Insecure SSH configuration demonstrated  ✅
Insecure SSH configuration never applied ✅
Final Terraform plan clean               ✅
Sensitive local artifacts ignored        ✅
```

## Conclusion

This project demonstrates a complete Infrastructure as Code security workflow rather than simply provisioning an EC2 instance.

The infrastructure was designed with Terraform, deployed to AWS, hardened using IAM, encrypted storage, IMDSv2, and Systems Manager, audited with Prowler, remediated through code, and independently verified after remediation.

The final architecture intentionally keeps some lab-level simplifications documented while demonstrating the core principles required for secure, auditable, and reproducible cloud infrastructure.
