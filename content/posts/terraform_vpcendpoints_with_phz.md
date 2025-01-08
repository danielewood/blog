---
author: ["Daniel Wood"]
title: "Terraform: Managing AWS VPC Endpoints and Private Hosted Zones"
date: "2025-01-07"
description: "A comprehensive guide to setting up VPC Endpoints with Private Hosted Zones in Terraform, addressing multi-VPC DNS resolution challenges"
summary: "Learn how to effectively manage AWS VPC Endpoints with Private Hosted Zones using Terraform, enabling DNS resolution across multiple VPCs while minimizing configuration complexity"
tags: ["opentofu", "terraform", "iac", "devops", "aws", "vpc", "dns"]
categories: ["infrastructure", "cloud", "tutorials", "terraform", "opentofu"]
series: ["terraform", "opentofu"]
ShowToc: true
TocOpen: true
---

Sometimes instead of a simple AWS VPC Endpoint, you need to control your own DNS through a Private Hosted Zone (PHZ). This is useful when you have multiple VPCs that need to use the same VPC Endpoints as you cannot share the AWS created DNS across VPCs/Accounts. In this guide, we'll walk through how to set up a VPC Endpoint with a Private Hosted Zone using Terraform.

## Challenge

AWS VPC Endpoint DNS is typically limited to the originating VPC. This means that if you have multiple VPCs that need to access the same VPC Endpoint, you can't share the DNS resolution across VPCs. This is a problem if you have a multi-account or multi-VPC setup where you want to share a VPC Endpoint across VPCs.

When you create a VPC Endpoint, and turn off the `private_dns_enabled` flag, you lose the ability to use the AWS provided DNS. You also are not given information about what DNS records you should have created in your Private Hosted Zone. Making assumptions here leads to a lot of misses and misconfigurations. Fortunately, the data is available in the AWS API, and we can use Terraform to extract it.

## Solution

Using `aws_vpc_endpoint_service` to create a dynamic configuration that handles all the multitude of DNS entries is a solution where we remove the guess work from this process. This is a relatively new ability, as the data object for this resource was missing the required `private_dns_names` until [recently](https://github.com/hashicorp/terraform-provider-aws/pull/39659) (Yes, that's my commit, you're welcome). Let's break it down step by step.

### 1. Define Your VPCes

First, define which VPC Endpoint Services you want to create:

```hcl
locals {
  shared_vpces = [
    "codeartifact.repositories",
  ]
}
```

### 2. Lookup and Create VPC Endpoints

Look up the details of each VPC desired endpoint and create the VPC Endpoint:

```hcl
data "aws_vpc_endpoint_service" "this" {
  for_each = toset(local.shared_vpces)

  service = each.value
}

resource "aws_vpc_endpoint" "this" {
  for_each = data.aws_vpc_endpoint_service.this

  service_name = each.value.service_name
  vpc_id       = data.aws_vpc.shared.id
  subnet_ids   = data.aws_subnets.shared.ids

  security_group_ids = [
    aws_security_group.shared.id
  ]

  vpc_endpoint_type   = "Interface"
  private_dns_enabled = false

  tags = {
    Name = each.value.service
  }
}
```

### 3. Some convoluted terraform processing and loops

Here's where the magic happens. We'll use nested `for_each` operations to create all possible dns entries and needed metadata for creating zones and records:

```hcl
locals {
  # Define the list of DNS zones to exclude from creation/association (["on.aws", "amazonaws.com"])
  dns_zone_exclusion = []

  # Parse private DNS names for VPC endpoint services, excluding any zone names in the dns_zone_exclusion list
  # This local block creates a list of parsed DNS entries for each VPC endpoint service.
  # Each DNS entry includes the key, the VPC endpoint name, the DNS name, a zone name without wildcard, and a flag for wildcard presence.
  shared_vpce_dns_zones = flatten([
    for vpce_name, vpce in data.aws_vpc_endpoint_service.this : [
      for private_dns_name in tolist(vpce.private_dns_names) : {
        key          = replace(vpce_name, ".", "-")
        vpce_name    = vpce_name
        dns_name     = private_dns_name
        zone_name    = startswith(private_dns_name, "*.") ? replace(private_dns_name, "*.", "") : private_dns_name
        has_wildcard = startswith(private_dns_name, "*.")
      }
      # Only include the entry if the zone_name is not in dns_zone_exclusion
      if !contains(local.dns_zone_exclusion, (startswith(private_dns_name, "*.") ? replace(private_dns_name, "*.", "") : private_dns_name))
    ]
  ])
}
```

### 4. Create the Private Hosted Zones for the VPC Endpoints

Here we will create our PHZ, this way we can selectively share the DNS entries across VPCs:

```hcl
resource "aws_route53_zone" "vpc_endpoint" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns }

  name = each.value.zone_name
  vpc {
    vpc_id = data.aws_vpc.shared.id
  }

  lifecycle {
    ignore_changes = [vpc]
  }

  tags = {
    VPCEndpoint = each.value.vpce_name
  }
}

resource "aws_route53_record" "vpc_endpoint" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns }

  zone_id = aws_route53_zone.vpc_endpoint[each.key].zone_id
  name    = aws_route53_zone.vpc_endpoint[each.key].name
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["dns_name"]
    zone_id                = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["hosted_zone_id"]
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "vpc_endpoint_wildcard" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns if dns.has_wildcard }

  zone_id = aws_route53_zone.vpc_endpoint[each.key].zone_id
  name    = "*.${aws_route53_zone.vpc_endpoint[each.key].name}"
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["dns_name"]
    zone_id                = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["hosted_zone_id"]
    evaluate_target_health = true
  }
}
```

## Results

When you apply this configuration, you'll see outputs confirming that terraform has successfully created VPC endpoints and PHZs for each service

Example output:
```
Plan: 8 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────

🟢 Create:
  + aws_route53_record.vpc_endpoint["codeartifact.us-west-2.on.aws"]
  + aws_route53_record.vpc_endpoint["d.codeartifact.us-west-2.amazonaws.com"]
  + aws_route53_record.vpc_endpoint_wildcard["codeartifact.us-west-2.on.aws"]
  + aws_route53_record.vpc_endpoint_wildcard["d.codeartifact.us-west-2.amazonaws.com"]
  + aws_route53_zone.vpc_endpoint["codeartifact.us-west-2.on.aws"]
  + aws_route53_zone.vpc_endpoint["d.codeartifact.us-west-2.amazonaws.com"]
  + aws_security_group.shared
  + aws_vpc_endpoint.this["codeartifact.repositories"]
```

## Complete example

I know in the end, we all just want a complete example to copy-paste. Here you go:

```hcl
provider "aws" {
  region = "us-west-2"
  alias  = "shared"
}

data "aws_vpc" "shared" {
  provider = aws.shared
}

data "aws_subnets" "shared" {
  provider = aws.shared
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  filter {
    name   = "tag:Name"
    values = ["S-*"]
  }
}

resource "aws_security_group" "shared" {
  provider    = aws.shared
  name        = "shared"
  description = "shared"
  vpc_id      = data.aws_vpc.shared.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow inbound HTTPS (TCP 443)"
  }

  tags = {
    Name = "shared"
  }
}

data "aws_region" "shared" {
  provider = aws.shared
}

locals {
  shared_vpces = [
    # "airflow.api",
    # "airflow.env",
    # "airflow.ops",
    # "application-autoscaling",
    # "athena",
    # "autoscaling",
    # "batch",
    # "bedrock-runtime",
    # "cloudformation",
    # "cloudtrail",
    # "codeartifact.api",
    "codeartifact.repositories",
    # "ec2",
    # "ec2messages",
    # "ecr.api",
    # "ecr.dkr",
    # "ecs",
    # "ecs-agent",
    # "ecs-telemetry",
    # "eks",
    # "elasticloadbalancing",
    # "events",
    # "execute-api",
    # "glue",
    # "kinesis-firehose",
    # "kms",
    # "lambda",
    # "logs",
    # "monitoring",
    # "rds",
    # "sagemaker.api",
    # "secretsmanager",
    # "sns",
    # "sqs",
    # "ssm",
    # "ssmmessages",
    # "sts",
  ]
}

data "aws_vpc_endpoint_service" "this" {
  for_each = toset(local.shared_vpces)

  provider = aws.shared

  service = each.value
}

resource "aws_vpc_endpoint" "this" {
  for_each = data.aws_vpc_endpoint_service.this
  provider = aws.shared

  service_name = each.value.service_name
  vpc_id       = data.aws_vpc.shared.id
  subnet_ids   = data.aws_subnets.shared.ids

  security_group_ids = [
    aws_security_group.shared.id
  ]

  vpc_endpoint_type   = "Interface"
  private_dns_enabled = false

  tags = {
    Name = each.value.service
  }
}

locals {
  dns_zone_exclusion = []

  # Parse private DNS names for VPC endpoint services, excluding any zone names in the dns_zone_exclusion list
  # This local block creates a list of parsed DNS entries for each VPC endpoint service.
  # Each DNS entry includes the key, the VPC endpoint name, the DNS name, a zone name without wildcard, and a flag for wildcard presence.
  shared_vpce_dns_zones = flatten([
    for vpce_name, vpce in data.aws_vpc_endpoint_service.this : [
      for private_dns_name in tolist(vpce.private_dns_names) : {
        key          = replace(vpce_name, ".", "-")
        vpce_name    = vpce_name
        dns_name     = private_dns_name
        zone_name    = startswith(private_dns_name, "*.") ? replace(private_dns_name, "*.", "") : private_dns_name
        has_wildcard = startswith(private_dns_name, "*.")
      }
      # Only include the entry if the zone_name is not in dns_zone_exclusion
      if !contains(local.dns_zone_exclusion, (startswith(private_dns_name, "*.") ? replace(private_dns_name, "*.", "") : private_dns_name))
    ]
  ])
}

resource "aws_route53_zone" "vpc_endpoint" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns }

  provider = aws.shared

  name = each.value.zone_name
  vpc {
    vpc_id = data.aws_vpc.shared.id
  }

  lifecycle {
    ignore_changes = [vpc]
  }

  tags = {
    VPCEndpoint = each.value.vpce_name
  }
}

resource "aws_route53_record" "vpc_endpoint" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns }

  provider = aws.shared

  zone_id = aws_route53_zone.vpc_endpoint[each.key].zone_id
  name    = aws_route53_zone.vpc_endpoint[each.key].name
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["dns_name"]
    zone_id                = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["hosted_zone_id"]
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "vpc_endpoint_wildcard" {
  for_each = { for dns in local.shared_vpce_dns_zones : dns.zone_name => dns if dns.has_wildcard }

  provider = aws.shared

  zone_id = aws_route53_zone.vpc_endpoint[each.key].zone_id
  name    = "*.${aws_route53_zone.vpc_endpoint[each.key].name}"
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["dns_name"]
    zone_id                = aws_vpc_endpoint.this[each.value.vpce_name].dns_entry[0]["hosted_zone_id"]
    evaluate_target_health = true
  }
}
```
