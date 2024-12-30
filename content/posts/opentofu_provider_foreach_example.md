---
author: ["Daniel Wood"]
title: "OpenTofu: Using for_each with Providers"
date: "2024-12-29"
description: "A guide to using the new for_each feature with providers in OpenTofu 1.9+, enabling dynamic multi-region infrastructure deployment."
summary: "Learn how to leverage OpenTofu 1.9's new provider for_each feature to simplify multi-region infrastructure management and reduce code duplication."
tags: ["opentofu", "terraform", "iac", "devops", "aws"]
categories: ["infrastructure", "cloud", "tutorials", "terraform", "opentofu"]
series: ["terraform", "opentofu"]
ShowToc: true
TocOpen: true
---

Managing infrastructure across multiple regions/profiles/roles has always been a pain point with Terraform. Until now, the only approach involved creating separate provider blocks for each variant, leading to large, repetitive code blocks. OpenTofu 1.9 brings a [much requested](https://github.com/hashicorp/terraform/issues/19932) feature to Terraform, `for_each` to provider configurations. This is one of the great value propositions of OpenTofu, features that have been pain points for the community are finally being addressed.

## The Old Way

Previously, supporting multiple regions/profiles/etc meant writing provider configurations that looked something like this:

```hcl
provider "aws" {
  alias  = "global"
  region = "us-east-1"
}

provider "aws" {
  alias  = "backup"
  region = "us-west-2"
}
```

Each new region required another provider block, and making changes meant updating multiple places in your code.

## Enter `for_each`

The new provider `for_each` feature transforms this repetitive pattern into something much more concise. Here's a practical example of setting up multi-region AWS infrastructure:

```hcl
provider "aws" {
  for_each = var.aws_regions
  alias    = "by_region"
  region   = each.value
}
```

OpenTofu creates a provider instance for each region in your map. The `alias` field creates a named reference point, while `each.value` pulls the actual region from your map.

```hcl
variable "aws_regions" {
  type = map(string)
  default = {
    "global" : "us-east-1",
    "backup" : "us-west-2"
  }
}
```

This map defines your regions with meaningful aliases. Want to add another region? Just add another entry to the map.

Now you can use regional resources with ease:

```hcl
data "aws_availability_zones" "this" {
  for_each = var.aws_regions
  provider = aws.by_region[each.key]
}
```

The `provider` reference uses the region alias as a lookup key, automatically matching each data source with its corresponding regional provider.

Want to see what regions you're working with? A "simple" output block does the trick:

```hcl
output "provider_regions" {
  value = { for k, v in data.aws_availability_zones.this : k => tolist(v.group_names)[0] }
}
```

Sometimes you need to work with just one provider for a resource. No problem:

```hcl
data "aws_availability_zones" "global" {
  provider = aws.by_region["global"]
}

output "global_region" {
  value = tolist(data.aws_availability_zones.global.group_names)[0]
}
```

Running this configuration shows your regions are properly set up:

```hcl
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

global_region = "us-east-1"
provider_regions = {
  "backup" = "us-west-2"
  "global" = "us-east-1"
}
```

The output confirms that both the global region and the complete region map are configured correctly. You can see how the aliases map to their respective AWS regions, making it easy to verify your setup.

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

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~>5"
    }
  }
}

provider "aws" {
  for_each = var.aws_regions
  alias    = "by_region"

  region = each.value
}

data "aws_availability_zones" "this" {
  for_each = var.aws_regions
  provider = aws.by_region[each.key]
}

output "provider_regions" {
  value = { for k, v in data.aws_availability_zones.this : k => tolist(v.group_names)[0] }
}

data "aws_availability_zones" "global" {
  provider = aws.by_region["global"]
}

output "global_region" {
  value = tolist(data.aws_availability_zones.global.group_names)[0]
}
```