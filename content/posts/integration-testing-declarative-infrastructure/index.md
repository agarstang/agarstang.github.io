---
title: "Why Integration Testing Frameworks Add Little Value for Declarative Infrastructure"
date: 2026-04-01
summary: "Terraform is declarative. What you write is the specification. So why write test suites that restate what the code already says?"
tags: ["terraform", "testing", "IaC", "devops"]
---

Terraform is declarative. What you write is the specification. The HCL you commit describes the desired end state, and Terraform's job is to make reality match. So why write test suites that restate what the code already says?

Tools like [Terratest](https://terratest.gruntwork.io/) and [Terraform's own test framework](https://developer.hashicorp.com/terraform/language/tests) do roughly the same thing:

1. Provision real infrastructure from your `.tf` files
2. Run assertions against the provisioned resources
3. Tear everything down

The intent is reasonable. But look at what you're actually asserting.

## The Tautology Problem

Given this Terraform:

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-app-data-bucket"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

The corresponding test:

```go
bucketID := terraform.Output(t, terraformOptions, "bucket_id")
aws.AssertS3BucketExists(t, region, bucketID)
aws.AssertS3BucketVersioningExists(t, region, bucketID)
```

What has this verified that the `.tf` file didn't already declare? You wrote "create a versioned S3 bucket" and then tested "does a versioned S3 bucket exist." That's tautology.

With imperative code, tests verify behaviour that isn't obvious from reading the source. A function might have edge cases or side effects that aren't self-evident. Declarative code doesn't have that problem; the whole point is that the code *is* the desired state.

## Most "Tests" Are Really Policy Checks

Dig into real world Terratest suites and a pattern shows up quickly. The majority of assertions aren't testing functionality, they're validating compliance:

- "Assert the S3 bucket has encryption enabled"
- "Assert the security group doesn't allow 0.0.0.0/0 on port 22"
- "Assert the RDS instance has storage encryption turned on"
- "Assert all resources have the required tags"

These aren't integration tests. They're policy checks dressed up as integration tests, and you're provisioning real infrastructure just to evaluate rules that could be checked statically.

## Policy Engines Already Do This

In Terraform Enterprise (TFE), you can use [Sentinel](https://www.hashicorp.com/sentinel) to validate policies. Sentinel policies evaluate against the plan *before* any infrastructure is created:

```python
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_server_side_encryption_configuration" and
  rc.mode is "managed"
}

main = rule {
  all s3_buckets as _, bucket {
    bucket.change.after.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm is "aws:kms"
  }
}
```

For regular Terraform or [OpenTofu](https://opentofu.org/) users without TFE, external policy checkers like [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) provide the same capability. OPA evaluates plan JSON against Rego policies, giving you identical pre-apply validation without vendor lock-in.

These run in seconds, cost nothing, and block non-compliant plans at the source.

Compare that to a Terratest suite that:

1. Spends 5-15 minutes provisioning real resources
2. Asserts the same encryption property against the live bucket
3. Tears everything down
4. Bills you for the infrastructure time
5. Can fail due to API rate limits or eventual consistency

Same answer ("is encryption configured correctly?") but through a much more expensive and fragile path. You're provisioning real infrastructure to enforce a policy that Sentinel catches at plan time without creating a single resource.

## When Integration Tests Are Justified

This isn't an argument against all testing. There are cases where provisioning real infrastructure makes sense:

- Multi-resource orchestration where wiring matters. A Lambda triggered by an SQS queue fed by an S3 event notification has failure modes that a plan can't reveal.
- Provider behaviour that isn't well documented or has known inconsistencies across regions.
- Shared modules consumed by many teams, where a periodic integration run can catch provider regressions.

None of these are "does the resource exist with the right tags." They're testing interactions that can't be inferred from the code alone.

## Keep Terraform Declarative

Part of why teams reach for testing frameworks is that their Terraform has stopped being purely declarative. Once you introduce `count` with conditionals, `for_each` over computed maps, heavy use of `locals` for string manipulation, or deeply nested dynamic blocks, the code starts behaving more like a program than a specification. You can no longer read the desired state directly from the file, so it feels like it needs tests.

The fix isn't to add tests. It's to simplify the Terraform.

- Prefer explicit resources over conditional ones.
- Avoid computing resource attributes from complex expressions. If a value needs three `locals` and a `join()` to produce, that complexity probably doesn't belong in Terraform at all.
- Use workspaces or variable files to handle environment differences rather than branching logic inside modules.
- Keep modules thin. If a module has more `variable` blocks than resource attributes, it's not simplifying anything.

When you can look at a `.tf` file and immediately see what will exist, you don't feel the urge to test it. The code reads as its own specification again.

## The Better Default

For most Terraform code:

1. `terraform validate` catches syntax and type errors instantly.
2. `terraform plan` (locally or in TFE) shows you what will change. A human reviews the diff.
3. `terraform apply` in a lower (dev/qa) env.
4. Sentinel policies in TFE enforce compliance and org policies before anything is provisioned.

These validate intent and compliance without provisioning infrastructure to confirm that a declarative engine did what it was told to do.

## Conclusion

The declarative model means the code is the test. Most Terratest suites, when you look at what they're actually checking, are enforcing policy. Sentinel handles that natively in TFE at a fraction of the cost. Save integration testing for genuinely complex resource interactions and let your policy engine do its job.
