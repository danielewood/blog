---
author: ["Daniel Wood"]
title: "OpenTofu: Complex Provider Configuration with for_each"
date: "2024-12-30"
description: "An advanced guide to using OpenTofu's provider for_each feature to manage multi-account, multi-role, multi-region AWS infrastructure"
summary: "Learn how to leverage OpenTofu 1.9's provider for_each feature to manage complex AWS infrastructure spanning multiple accounts, roles, and regions while minimizing code duplication"
tags: ["opentofu", "terraform", "iac", "devops", "aws"]
categories: ["infrastructure", "cloud", "tutorials", "terraform", "opentofu"]
series: ["terraform", "opentofu"]
ShowToc: true
TocOpen: true
---

In my [previous post](../opentofu_provider_foreach_example), we explored OpenTofu 1.9's new provider `for_each` feature for managing multi-region infrastructure. Today, we'll take this concept further by managing infrastructure across multiple AWS accounts and IAM roles, in addition to regions. This is a common requirement in enterprise environments where you need to manage resources across development, staging, and production accounts, with different IAM roles for varying levels of access.

## Challenge

Enterprise AWS environments typically involve:
- Multiple AWS accounts (dev, qa, prod)
- Different IAM roles for various access levels (admin, reader)
- Multiple regions for redundancy
- Need to assume different roles in different accounts

Without `for_each`, this would require manually writing provider configurations for every combination of account, role, and region. For just 2 accounts, 2 roles, and 2 regions, that's already 8 provider blocks!

## Solution

Using OpenTofu's provider `for_each`, we can create a dynamic configuration that handles all these combinations elegantly. Let's break it down step by step.

### 1. Define Your Variables

First, define your regions and account IDs:

```hcl
variable "aws_regions" {
  type = map(string)
  default = {
    "global" : "us-east-1",
    "backup" : "us-west-2"
  }
}

variable "aws_account_ids" {
  type = map(string)
  default = {
    "dev" : "123456789012"
    "qa"  : "234567890123"
  }
}
```

### 2. Define Role Configurations

Set up your role definitions in a local variable:

```hcl
locals {
  role_definitions = {
    "admin"  = "admin-role"
    "reader" = "reader-role"
  }
}
```

### 3. Create Provider Configurations

Here's where the magic happens. We'll use nested `for_each` operations to create all possible combinations:

```hcl
locals {
  # Create account + role combinations
  account_role_combinations = merge([
    for account_name, account_id in var.aws_account_ids : {
      for role_name, role_path in local.role_definitions :
      "${account_name}_${role_name}" => "arn:aws:iam::${account_id}:role/${role_path}"
    }
  ]...)

  # Create final provider configurations combining account_roles with regions
  provider_configurations = merge([
    for account_role, role_arn in local.account_role_combinations : {
      for region_name, region in var.aws_regions :
      "${account_role}_${region_name}" => {
        role_arn = role_arn
        region   = region
      }
    }
  ]...)
}
```

### 4. Configure the Provider

With our configurations ready, we can create the provider block:

```hcl
provider "aws" {
  for_each = local.provider_configurations
  alias    = "this"

  region = each.value.region
  assume_role {
    role_arn = each.value.role_arn
  }
}
```

### 5. Validate the Configuration

Let's add some data sources and outputs to verify everything works:

```hcl
data "aws_availability_zones" "all_zones" {
  for_each = local.provider_configurations
  provider = aws.this[each.key]
}

data "aws_caller_identity" "assumed_roles" {
  for_each = local.provider_configurations
  provider = aws.this[each.key]
}

output "availability_zones" {
  description = "Available AZs for each provider configuration"
  value = {
    for k, v in data.aws_availability_zones.all_zones : k => tolist(v.group_names)[0]
  }
}

output "assumed_role_arns" {
  description = "Assumed role ARN for each provider configuration"
  value = {
    for k, v in data.aws_caller_identity.assumed_roles : k => v.arn
  }
}
```

## Results

When you apply this configuration, you'll see outputs confirming that OpenTofu has successfully:
- Created provider configurations for all combinations
- Assumed the correct roles in each account
- Accessed the correct regions

Example output:
```
assumed_role_arns = {
  "dev_admin_backup"  = "arn:aws:sts::123456789012:assumed-role/admin-role/..."
  "dev_admin_global"  = "arn:aws:sts::123456789012:assumed-role/admin-role/..."
  "dev_reader_backup" = "arn:aws:sts::123456789012:assumed-role/reader-role/..."
  "dev_reader_global" = "arn:aws:sts::123456789012:assumed-role/reader-role/..."
  "qa_admin_backup"   = "arn:aws:sts::234567890123:assumed-role/admin-role/..."
  "qa_admin_global"   = "arn:aws:sts::234567890123:assumed-role/admin-role/..."
  "qa_reader_backup"  = "arn:aws:sts::234567890123:assumed-role/reader-role/..."
  "qa_reader_global"  = "arn:aws:sts::234567890123:assumed-role/reader-role/..."
}
```

## Benefits of This Approach

1. **Maintainability**: Adding new accounts, roles, or regions is as simple as adding entries to the respective maps.
2. **DRY Code**: No repetition of provider blocks, even with many combinations.
3. **Validation**: Easy to verify that all configurations are working through the outputs.
4. **Flexibility**: Easy to modify role paths or add new configurations without changing the core structure.

## Real World Usage

This pattern is particularly useful when:
- Managing shared infrastructure across multiple AWS accounts
- Implementing least-privilege access patterns
- Setting up disaster recovery across regions
- Managing development, staging, and production environments

## Conclusion

OpenTofu's provider `for_each` feature transforms what would have been dozens of repetitive provider blocks into a clean, maintainable configuration. This approach scales well as your infrastructure grows, making it easier to manage complex, multi-account AWS environments.

## Complete example

I know in the end, we all just want a complete example to copy-paste. Here you go:

```hcl
variable "aws_regions" {
  type = map(string)
  default = {
    "global" : "us-east-1",
    "backup" : "us-west-2"
  }
}

variable "aws_account_ids" {
  type = map(string)
  default = {
    "dev" : "123456789012"
    "qa"  : "234567890123"
  }
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~>5"
    }
  }
}

locals {
  role_definitions = {
    "admin" = "admin-role"
    "reader" = "reader-role"
  }

  # Create account + role combinations
  account_role_combinations = merge([
    for account_name, account_id in var.aws_account_ids : {
      for role_name, role_path in local.role_definitions :
      "${account_name}_${role_name}" => "arn:aws:iam::${account_id}:role/${role_path}"
    }
  ]...)

  # Create final provider configurations combining account_roles with regions
  provider_configurations = merge([
    for account_role, role_arn in local.account_role_combinations : {
      for region_name, region in var.aws_regions :
      "${account_role}_${region_name}" => {
        role_arn = role_arn
        region   = region
      }
    }
  ]...)
}

provider "aws" {
  for_each = local.provider_configurations
  alias    = "this"

  region = each.value.region
  assume_role {
    role_arn = each.value.role_arn
  }
}

data "aws_availability_zones" "all_zones" {
  for_each = local.provider_configurations
  provider = aws.this[each.key]
}

data "aws_caller_identity" "assumed_roles" {
  for_each = local.provider_configurations
  provider = aws.this[each.key]
}

output "availability_zones" {
  description = "Available AZs for each provider configuration"
  value = {
    for k, v in data.aws_availability_zones.all_zones : k => tolist(v.group_names)[0]
  }
}

output "assumed_role_arns" {
  description = "Assumed role ARN for each provider configuration"
  value = {
    for k, v in data.aws_caller_identity.assumed_roles : k => v.arn
  }
}
```