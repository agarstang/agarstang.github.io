---
title: "The Pitfalls of Overusing Modules in Terraform"
date: 2025-10-22
summary: "Terraform modules are a powerful way to organize and reuse infrastructure-as-code, but overusing them can lead to unnecessary complexity."
tags: ["terraform", "IaC", "devops", "AWS"]
---

[Terraform](https://www.terraform.io/) modules are a powerful way to organize and reuse infrastructure-as-code, but overusing them can lead to unnecessary complexity.

One of Terraform's strengths is its declarative nature; it's easy to understand what resources are being created just by reading the code. Overusing modules obscures this visibility. Instead of seeing a clear set of resources, engineers must dig through multiple layers of abstraction just to determine what's being deployed.

## Increased Complexity Without Simplification

To make a module truly reusable, you often have to expose a wide range of input variables to cover different use cases. This means nearly every attribute of every resource needs to be configurable. At this point, the module's complexity is equal to defining the resources themselves, but without the direct readability and maintainability of straightforward Terraform configurations. Instead of simplifying infrastructure, excessive modularization can create an unnecessary abstraction layer that doesn't add real value.

For example, consider the following module instantiation:

```hcl
module "ec2_instance" {
  source = "./modules/ec2"
  instance_type = "t2.micro"
  ami           = "ami-123456"
  key_name      = "my-key"
  vpc_security_group_ids = ["sg-123456"]
  subnet_id     = "subnet-123456"
  tags = {
    Name = "example-instance"
  }
}
```

That, in this case, deploys a couple of resources:

```hcl
resource "aws_instance" "this" {
  instance_type = var.instance_type
  ami           = var.ami
  key_name      = var.key_name
  vpc_security_group_ids = concat(var.vpc_security_group_ids, [aws_security_group.allow_icmp.id])
  subnet_id     = var.subnet_id
  tags          = var.tags
}

resource "aws_security_group" "allow_icmp" {
  name        = "allow_icmp"
  description = "Allow icmp ping traffic"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = -1
    to_port         = -1
    protocol        = "icmp"
    cidr_blocks     = ["0.0.0.0/0"]
  }

  tags = var.tags

}
```

***💡 The number of exposed variables is almost equal to the number of attributes on the resource(s).***

```hcl
variable "instance_type" {}
variable "ami" {}
variable "key_name" {}
variable "vpc_security_group_ids" {}
variable "subnet_id" {}
variable "vpc_id" {}
variable "tags" {}
```

If all the attributes are exposed as variable there's very little simplification, only an additional layer of abstraction. Public Terraform modules often suffer from this.

## Debugging Can Be Difficult

When issues arise, tracing problems through deeply nested modules is a headache. Instead of directly inspecting a resource's configuration in a simple Terraform file, engineers must navigate through layers of abstraction, tracking variable inheritance and module outputs to understand what the actual values are.

## Slower Iteration and Customization

Generic modules often require additional effort to extend or modify compared to directly updating resource definitions. If a team needs a small adjustment, they may need extended efforts to maintain the modules existing contract and compatibility. This slows down infrastructure evolution and can lead to modules supporting team's edge cases.

💡If the Terraform is just for your team don't commit to contracts you may then need to maintain indefinitely.

## When Should You Use Modules?

While overusing modules is problematic, they are still valuable in certain scenarios:

- **Encapsulating common patterns** – If your team frequently deploys identical sets of resources (e.g., a standard 3-tier app).

- **Enforcing best practices** – Modules can provide guardrails to ensure security policies or tagging standards are consistently applied. In fact abstracting away these attributes can significantly reduce the inherent complexity of resources.

- **Abstracting complex configurations** – When a set of resources requires intricate relationships and dependencies, a module can encapsulate that. For example this can be useful for a Global RDS where you need to deploy the same or similar resources repeatedly across regions; often doing this from a single workspace that exists in the primary region.

***Terraform modules are a double-edged sword.*** They enable reusability and standardization; while overusing them (especially for basic infrastructure) can create unnecessary abstraction, reduce readability, and make debugging hard.

## Links

- [Terraform Docs](https://developer.hashicorp.com/terraform?product_intent=terraform)
- [Resourcely - 10 Terraform Config Root Setups](https://www.resourcely.io/post/10-terraform-config-root-setups)
