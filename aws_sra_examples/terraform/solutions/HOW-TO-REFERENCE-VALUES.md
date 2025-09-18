# How to Reference Values in AWS SRA Terraform Configuration

This guide explains how to properly reference and configure values when deploying the AWS Security Reference Architecture (SRA) using Terraform.

## Overview

The AWS SRA Terraform edition uses a two-step process for configuration:

1. **Common Prerequisites**: Creates SSM parameters, S3 buckets, and other shared resources
2. **Solutions Deployment**: Uses the values from step 1 to deploy security services

## Configuration Files

### backend.tfvars
This file contains Terraform backend configuration for remote state storage:

```hcl
bucket         = "sra-terraform-state-bucket-123456789012-us-east-1"
key            = "state/sra_state.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "sra-tfstate-lock"
```

**How values are populated:**
- `bucket`: Auto-generated during common prerequisites deployment
- `region`: Retrieved from `/sra/control-tower/home-region` SSM parameter
- `dynamodb_table`: Created by the DynamoDB module

### config.tfvars
This file contains all configuration variables for the SRA solutions:

```hcl
# Main configuration (auto-populated from SSM parameters)
audit_account_id           = "111111111111"
management_account_id      = "333333333333"
home_region               = "us-east-1"

# Service enablement (user configurable)
enable_gd                 = true
enable_sh                 = true
enable_access_analyzer    = true
```

## How to Reference Values

### 1. Account IDs and Organization Information

Values are automatically populated from SSM parameters created during common prerequisites:

- **audit_account_id**: Retrieved from `/sra/control-tower/audit-account-id`
- **management_account_id**: Retrieved from `/sra/control-tower/management-account-id`
- **log_archive_account_id**: Retrieved from `/sra/control-tower/log-archive-account-id`
- **organization_id**: Retrieved from `/sra/control-tower/organization-id`

### 2. Regional Configuration

- **home_region**: Primary Control Tower region from `/sra/control-tower/home-region`
- **customer_control_tower_regions**: Comma-separated list of managed regions
- **enabled_regions**: Additional regions (leave empty for Control Tower environments)

### 3. Service Configuration

Each service can be enabled/disabled and configured independently:

#### GuardDuty Example:
```hcl
enable_gd                            = true
enable_s3_logs                       = true
enable_malware_protection            = true
finding_publishing_frequency         = "FIFTEEN_MINUTES"
guardduty_control_tower_regions_only = true
```

#### Security Hub Example:
```hcl
enable_sh                            = true
enable_nist_standard                 = true
enable_security_best_practices_standard = true
compliance_frequency                 = 7
```

### 4. Using Variables in Terraform Modules

Within the Terraform modules, values are referenced as:

```hcl
module "guard_duty" {
  count = var.enable_gd ? 1 : 0
  
  audit_account_id      = local.audit_account_id
  management_account_id = local.management_account_id
  organization_id       = var.organization_id
  
  enable_s3_logs               = var.enable_s3_logs
  finding_publishing_frequency = var.finding_publishing_frequency
}
```

## Deployment Process

### Step 1: Deploy Common Prerequisites
```bash
cd aws_sra_examples/terraform/common
terraform init
terraform apply
```

This creates:
- SSM parameters with account and organization information
- S3 bucket for Terraform state
- DynamoDB table for state locking
- `backend.tfvars` and `config.tfvars` files

### Step 2: Configure Services
Edit the generated `config.tfvars` file to enable desired services:

```hcl
# Enable GuardDuty
enable_gd = true

# Enable Security Hub with NIST standard
enable_sh = true
enable_nist_standard = true

# Enable IAM Access Analyzer
enable_access_analyzer = true
```

### Step 3: Deploy Solutions
```bash
cd aws_sra_examples/terraform/solutions
python3 terraform_stack.py init
python3 terraform_stack.py apply
```

## Common Configuration Patterns

### Enable Basic Security Services:
```hcl
enable_gd              = true
enable_sh              = true
enable_access_analyzer = true
enable_cloudtrail_org  = true
```

### Enable Comprehensive Security Suite:
```hcl
enable_gd                            = true
enable_sh                            = true
enable_access_analyzer               = true
enable_macie                         = true
enable_inspector                     = true
enable_cloudtrail_org               = true
enable_iam_password_policy          = true

# Enable Security Hub standards
enable_security_best_practices_standard = true
enable_nist_standard                   = true
enable_cis_standard                    = true
```

### Control Tower vs. Organizations Configuration:

For **Control Tower** environments (default):
```hcl
# Values auto-populated from SSM parameters
customer_control_tower_regions = "us-east-1,us-west-2"
enabled_regions = ""
```

For **Organizations-only** environments:
```hcl
# Must be manually configured in common/variables.tf
control_tower = "false"
governed_regions = "us-east-1,us-west-2,eu-west-1"
security_account_id = "111111111111"
log_archive_account_id = "222222222222"
```

## Troubleshooting Configuration Issues

### Missing Configuration Files
If `backend.tfvars` or `config.tfvars` are missing:
1. Ensure common prerequisites deployed successfully
2. Check that all required SSM parameters exist
3. Re-run `terraform apply` in the common directory

### Invalid Variable Values
If you encounter validation errors:
1. Check that account IDs are 12 digits
2. Verify region names are valid AWS regions
3. Ensure boolean values use `true`/`false` (not strings)

### Service Dependencies
Some services have dependencies:
- Security Hub requires Config to be enabled
- Detective requires GuardDuty to be enabled first
- Access Analyzer requires delegated administrator registration

## Security Considerations

- **State File Security**: Terraform state is encrypted and stored in S3
- **Parameter Store**: SSM parameters may contain sensitive data
- **Cross-Account Access**: Ensure proper IAM roles are configured
- **Region Restrictions**: Only deploy to regions where services are available

## Next Steps

1. Review the generated `config.tfvars.example` file for all available options
2. Customize service configurations based on your security requirements
3. Test deployments in a non-production environment first
4. Monitor AWS costs as some services incur charges
5. Review AWS SRA documentation for security best practices