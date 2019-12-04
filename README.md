# terraform-aws-data-lake

_Note: Data Lakes is currently in Limited Availability._

Terraform modules which create AWS resources for a Segment Data Lake.

# Prerequisites

* Accept the [Data Lakes Terms of Service](https://app.segment.com/{workspace_slug}/destinations/catalog?category=DataLakes) (replace the `{workspace_slug}` with your workspace slug).
* Authorized [AWS account](https://aws.amazon.com/account/).
* Ability to run Terraform with your AWS Account. You must use Terraform 0.11 or higher.
* A subnet within a VPC for the EMR cluster to run in.
* [S3 Bucket](https://github.com/terraform-aws-modules/terraform-aws-s3-bucket) to send data from Segment to and to store logs.

## VPC

You'll need to provide a subnet within a VPC for the EMR to cluster to run in. Here are some resources that can guide you through setting up a VPC for your EMR cluster:
* https://aws.amazon.com/blogs/big-data/launching-and-running-an-amazon-emr-cluster-inside-a-vpc/
* https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-clusters-in-a-vpc.html
* https://github.com/terraform-aws-modules/terraform-aws-vpc

# Modules

The repository is split into multiple modules, and each can be used independently: 
* [iam](/modules/iam) - IAM roles that give Segment access to your AWS resources.
* [glue](/modules/glue) - Glue tables that Segment can write metadata to.
* [emr](/modules/emr) - EMR cluster that Segment can submit jobs to load events into your Data Lake.

# Usage

```
provider "aws" {
  region = "us-west-2" # Replace this with the AWS region your infrastructure is set up in.
}

locals {
  tags = {
    s3_bucket_name = "..."          # Replace this with the S3 bucket name to store your data.
    external_ids   = ["...", "..."] # Replace this with a list of your Segment sources that will be enabled for Data Lakes.
    subnet_id      = "..."          # Replace this with the subnet ID you want the EMR cluster to run in.
  }
}

module "glue" {
  source = "git@github.com:segmentio/terraform-aws-data-lake//modules/glue?ref=v0.1.0"

  name        = "segment_data_lake"
  description = "Glue DB for Segment Data Lake"
}

module "iam" {
  source = "git@github.com:segmentio/terraform-aws-data-lake//modules/iam?ref=v0.1.0"

  name               = "segment-data-lake-iam-role"
  s3_bucket          = "${local.s3_bucket_name}"
  glue_database_name = "segment_data_lake"
  external_ids       = "${local.external_ids}"
}

module "emr" {
  source = "git@github.com:segmentio/terraform-aws-data-lake//modules/emr?ref=v0.1.0"

  s3_bucket = "${local.s3_bucket_name}"
  subnet_id = "${local.subnet_id}"
}
```

With the Terraform CLI, you can run `terraform plan` to preview the changes by the modules, and `terraform apply` to generate the resources.

Note that creating the EMR cluster can take a while (typically 5 minutes).

Once applied, make a note of the following (you'll need to provide this information to your Segment contact):
* Segment source IDs and slugs that should be connect to the data lake
* The AWS Region and AWS Account ID where your Data Lake was setup.
* The ID of the generated EMR Cluster
* The name of the generate Glue database
* The ARN of the generated IAM role

# Common Errors

## The VPC/subnet configuration was invalid: No route to any external sources detected in Route Table for Subnet

```
Error: Error applying plan:
1 error(s) occurred:
* module.emr.aws_emr_cluster.segment_data_lake_emr_cluster: 1 error(s) occurred:
* aws_emr_cluster.segment_data_lake_emr_cluster: Error waiting for EMR Cluster state to be "WAITING" or "RUNNING": TERMINATED_WITH_ERRORS: VALIDATION_ERROR: The VPC/subnet configuration was invalid: No route to any external sources detected in Route Table for Subnet: subnet-{id} for VPC: vpc-{id}
Terraform does not automatically rollback in the face of errors.
Instead, your Terraform state file has been partially updated with
any resources that successfully completed. Please address the error
above and apply again to incrementally change your infrastructure.
exit status 1
```

The EMR cluster requires a route table attached to the subnet with an internet gateway. You can follow [this guide](https://aws.amazon.com/blogs/big-data/launching-and-running-an-amazon-emr-cluster-inside-a-vpc/) to 

## The subnet configuration was invalid: The subnet subnet-{id} does not exist.

```
Error: Error applying plan:
1 error(s) occurred:
* module.emr.aws_emr_cluster.segment_data_lake_emr_cluster: 1 error(s) occurred:
* aws_emr_cluster.segment_data_lake_emr_cluster: Error waiting for EMR Cluster state to be "WAITING" or "RUNNING": TERMINATED_WITH_ERRORS: VALIDATION_ERROR: The subnet configuration was invalid: The subnet subnet-{id} does not exist.
Terraform does not automatically rollback in the face of errors.
Instead, your Terraform state file has been partially updated with
any resources that successfully completed. Please address the error
above and apply again to incrementally change your infrastructure.
exit status 1
```

The EMR cluster requires a subnet with a VPC. You can follow [this guide](https://aws.amazon.com/blogs/big-data/launching-and-running-an-amazon-emr-cluster-inside-a-vpc/) to create a subnet.

# Supported Terraform Versions

Terraform 0.11 or higher is supported.

# Development

To develop in this repository, you'll want the following tools set up:

* [Terraform](https://www.terraform.io/downloads.html), >= 0.12 (note that 0.12 is used to develop this module, even though 0.11 is supported)
* [terraform-docs](https://github.com/segmentio/terraform-docs)
* [tflint](https://github.com/terraform-linters/tflint)
* [Ruby](https://www.ruby-lang.org/en/documentation/installation/), [>= 2.4.2](https://rvm.io)
* [Bundler](https://bundler.io)

To run unit tests, you also need an AWS account to be able to provision resources.

# License

Released under the [MIT License](https://opensource.org/licenses/MIT).
