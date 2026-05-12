# TechCorp AWS Infrastructure

## Architecture Overview

The infrastructure includes:
- VPC with public and private subnets across 2 availability zones
- Internet Gateway and NAT Gateways for network connectivity
- Bastion host for secure administrative access
- 2 Web servers running Apache behind an Application Load Balancer
- 1 Database server running PostgreSQL
- Proper security groups for network isolation

## Prerequisites

Before you begin, ensure you have the following:

**AWS Account** with appropriate permissions
**AWS CLI** installed and configured
   ```bash
   aws configure
   ```
**Terraform** installed (version >= 1.0)
   ```bash
   # Download from https://www.terraform.io/downloads
   terraform --version
   ```
**SSH Key Pair** created in your AWS region
   ```bash
   # Create a key pair in AWS Console or via CLI:
   aws ec2 create-key-pair --key-name techcorp-key --query 'KeyMaterial' --output text > techcorp-key.pem
   chmod 400 techcorp-key.pem
   ```
**Your Public IP Address**
   ```bash
   # Find your IP:
   curl ifconfig.me
   ```

## Project Structure

```
terraform-assessment/
├── main.tf                          # Main Terraform configuration
├── variables.tf                     # Variable declarations
├── outputs.tf                       # Output definitions
├── terraform.tfvars.example         # Example variables file
├── user_data/
│   ├── web_server_setup.sh         # Web server initialization script
│   └── db_server_setup.sh          # Database server initialization script
└── README.md                        # This file
```

## Setup Instructions

### Clone and Configure

Copy the example variables file:
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   ```

Edit `terraform.tfvars` with your actual values:
   ```hcl
   aws_region    = "us-east-1"
   my_ip         = "YOUR.PUBLIC.IP.ADDRESS/32"  # Replace with your IP
   key_pair_name = "techcorp-key"                # Your AWS key pair name
   server_password = "YourSecurePassword123!"    # Change this!
   ```

### Initialize Terraform

```bash
# Initialize Terraform and download providers
terraform init
```

### Review the Plan

```bash
# See what Terraform will create
terraform plan
```

This will show you all the resources that will be created. Review carefully!

### Deploy Infrastructure

```bash
# Deploy the infrastructure
terraform apply
```

Type `yes` when prompted to confirm.

**Note:** Deployment takes approximately 5-10 minutes.

### Get Outputs

```bash
# View important information about your deployment
terraform output
```

Save these outputs! You'll need them to access your infrastructure.

## Accessing Your Infrastructure

### Access Bastion Host

**Option A: Using SSH Key**
```bash
ssh -i techcorp-key.pem ec2-user@<BASTION_PUBLIC_IP>
```

**Option B: Using Password**
```bash
ssh techcorp@<BASTION_PUBLIC_IP>
# Password: The one you set in terraform.tfvars
```

### Access Web Servers from Bastion

Once on the bastion host:

```bash
# SSH to web server 1
ssh techcorp@<WEB_SERVER_1_PRIVATE_IP>
# Password: TechCorp2024!Secure (or your configured password)

# SSH to web server 2
ssh techcorp@<WEB_SERVER_2_PRIVATE_IP>
```

### Access Database Server from Bastion

```bash
# SSH to database server
ssh techcorp@<DATABASE_PRIVATE_IP>

# Once on the database server, test PostgreSQL
psql -h localhost -U techcorp_user -d techcorp_db
# Password: TechCorp2024!DB

# Run a test query
SELECT * FROM users;
```

### Access Web Application

Open your browser and visit:
```
http://<LOAD_BALANCER_DNS_NAME>
```

The page will show which instance is serving your request.

## Testing the Infrastructure

### Test 1: Verify Load Balancer

```bash
# The ALB should distribute traffic between both web servers
# Refresh the page multiple times to see different instance IDs
curl http://<LOAD_BALANCER_DNS_NAME>
```

### Test 2: Verify Database Connectivity

From bastion, SSH to a web server, then test database connection:

```bash
# Install PostgreSQL client on web server
sudo yum install -y postgresql

# Connect to database server
psql -h <DATABASE_PRIVATE_IP> -U techcorp_user -d techcorp_db

# Test query
SELECT * FROM users;
```

### Test 3: Verify High Availability

The infrastructure spans 2 availability zones:
- Public subnets in 2 AZs
- Private subnets in 2 AZs
- Web servers distributed across AZs
- NAT Gateways in both AZs

## Important Security Notes

**Security Considerations:**

**SSH Access**: Bastion host only accepts SSH from your IP address
**Password Authentication**: Enabled for ease of use (can be disabled for production)
**Database Access**: Only accessible from web servers and bastion
**Web Access**: Load balancer is publicly accessible on ports 80/443
**Secrets**: Never commit `terraform.tfvars` or `terraform.tfstate` to version control

## Troubleshooting

### Issue: Cannot SSH to Bastion
- Verify your IP in terraform.tfvars matches your current IP
- Check security group rules in AWS Console
- Verify key pair permissions: `chmod 400 techcorp-key.pem`

### Issue: Cannot Access Web Application
- Wait 5 minutes after deployment for instances to fully initialize
- Check target group health in AWS Console (EC2 → Target Groups)
- Verify security groups allow HTTP traffic

### Issue: Database Connection Fails
- Ensure you're connecting from web server or bastion
- Check PostgreSQL is running: `sudo systemctl status postgresql`
- Verify pg_hba.conf allows connections from VPC

### View User Data Logs
```bash
# On any EC2 instance, check user-data execution logs
sudo cat /var/log/cloud-init-output.log
sudo cat /var/log/user-data.log
```

## Monitoring Resources

### Check Resource Status

```bash
# VPC and Subnets
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=techcorp-vpc"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<VPC_ID>"

# EC2 Instances
aws ec2 describe-instances --filters "Name=tag:Name,Values=techcorp-*"

# Load Balancer
aws elbv2 describe-load-balancers --names techcorp-web-alb

# Target Group Health
aws elbv2 describe-target-health --target-group-arn <TARGET_GROUP_ARN>
```

## Cleanup Instructions

**Important:** This will destroy all resources and cannot be undone!

### Option 1: Terraform Destroy

```bash
# Destroy all resources
terraform destroy
```

Type `yes` when prompted.

### Option 2: Manual Cleanup (if Terraform fails)

If `terraform destroy` fails, manually delete in this order:
EC2 Instances
Load Balancer
Target Groups
NAT Gateways
Elastic IPs
Internet Gateway
Subnets
VPC

### Verify Cleanup

```bash
# Check for remaining resources
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=techcorp-vpc"
```

**Tip:** Run `terraform destroy` when not in use to avoid charges!

## Support

For issues or questions:
Check the Troubleshooting section above
Review Terraform error messages carefully
Check AWS CloudWatch logs for instance initialization errors
Verify all prerequisites are met

