# terraform-aws-data-lake

_Note: Data Lakes is currently in Limited Availability._

Terraform modules which create AWS resources for a Segment Data Lake.

# Prerequisites

* Accept the [Data Lakes Terms of Service](https://app.segment.com/{workspace_slug}/destinations/catalog?category=DataLakes) (replace the `{workspace_slug}` with your workspace slug).
* Authorized [AWS account](https://aws.amazon.com/account/).
* Ability to run Terraform with your AWS Account. You must use Terraform 0.11 or higher.
* A subnet within a VPC for the EMR cluster to run in.
* An [S3 Bucket](https://github.com/terraform-aws-modules/terraform-aws-s3-bucket) for Segment to load data into. You can create a new one just for this, or re-use an existing one you already have.

## VPC

You'll need to provide a subnet within a VPC for the EMR to cluster to run in. Here are some resources that can guide you through setting up a VPC for your EMR cluster:
* https://aws.amazon.com/blogs/big-data/launching-and-running-an-amazon-emr-cluster-inside-a-vpc/
* https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-clusters-in-a-vpc.html
* https://github.com/terraform-aws-modules/terraform-aws-vpc

# Modules

The repository is split into multiple modules, and each can be used independently:
* [iam](/modules/iam) - IAM roles that give Segment access to your AWS resources.
* [emr](/modules/emr) - EMR cluster that Segment can submit jobs to load events into your Data Lake.
* [glue](/modules/glue) - Glue tables that Segment can write metadata to.

# Usage

## Terraform Installation
*Note*  - Skip this section if you already have a working Terraform setup
### OSX:
`brew` on OSX should install the latest version of Terraform.
```
brew install terraform
```

### Centos/Ubuntu:
* Follow instructions [here](https://phoenixnap.com/kb/how-to-install-terraform-centos-ubuntu) to install on Centos/Ubuntu OS.
* Ensure that the version installed in > 0.11.x

Verify installation works by running:
```
terraform help
```

## Set up Project
* Create project directory
```
mkdir segment-datalakes-tf
```
* Create `main.tf` file
    * Update the `segment_sources` variable in the `locals` to the sources you want to sync 
    * Update the `name` in the `aws_s3_bucket` resource to the desired name of your S3 bucket
    * Update the `subnet_id` in the `emr` module to the subnet in which to create the EMR cluster

```hcl
provider "aws" {
  region = "us-west-2"  # Replace this with the AWS region your infrastructure is set up in.
}

locals {
  segment_sources = {
    # Find these in the Segment UI: (for each source you intend to connect)
    #  - Settings > SQL Settings > Schema Name (aka: Source Slug)
    #  - Settings > API Keys > Source ID
    <Segment Source Slug> = "<Segment Source ID>"
  }
}

# This is the target where Segment will write your data.
# You can skip this if you already have an S3 bucket and just reference that name manually later.
resource "aws_s3_bucket" "segment_datalake_s3" {
  name = "my-first-segment-datalake"
}

# Creates the IAM Policy that allows Segment to access the necessary resources
# in your AWS account for loading your data.
module "iam" {
  source = "git@github.com:segmentio/terraform-aws-data-lake//modules/iam?ref=v0.2.0"

  name         = "segment-data-lake-iam-role"
  s3_bucket    = "${aws_s3_bucket.segment_datalake_s3.name}"
  external_ids = "${values(local.segment_sources)}"
}

# Creates an EMR Cluster that Segment uses for performing the final ETL on your
# data that lands in S3.
module "emr" {
  source = "git@github.com:segmentio/terraform-aws-data-lake//modules/emr?ref=v0.2.0"

  s3_bucket = "${aws_s3_bucket.segment_datalake_s3.name}"
  subnet_id = "subnet-XXX" # Replace this with the subnet ID you want the EMR cluster to run in.
 
  # LEAVE THIS AS-IS
  iam_emr_autoscaling_role = "${module.iam.iam_emr_autoscaling_role}"
  iam_emr_service_role     = "${module.iam.iam_emr_service_role}"
  iam_emr_instance_profile = "${module.iam.iam_emr_instance_profile}"
}
```
## Provision Resources
* Provide AWS credentials of the account being used. More details here: https://www.terraform.io/docs/providers/aws/index.html
  ```
  export AWS_ACCESS_KEY_ID="anaccesskey"
  export AWS_SECRET_ACCESS_KEY="asecretkey"
  export AWS_DEFAULT_REGION="us-west-2"
  ```
* Initialize the references modules
  ```
  terraform init
  ```
  You should see a success message once you run the plan:
  ```
  Terraform has been successfully initialized!
  ```
* Run plan
  This does not create any resources. It just outputs what will be created after you run apply(next step).
  ```
  terraform plan
  ```
  You should see something like towards the end of the plan:
  ```
  Plan: 13 to add, 0 to change, 0 to destroy.
  ```
* Run apply - this step creates the resources in your AWS infrastructure
  ```
  terraform apply
  ```
  You should see:
  ```
  Apply complete! Resources: 13 added, 0 changed, 0 destroyed.
  ```

Note that creating the EMR cluster can take a while (typically 5 minutes).

Once applied, make a note of the following (you'll need to enter these as settings when configuring the Data Lake):
* The **AWS Region** and **AWS Account ID** where your Data Lake was configured
* The **Source ID and Slug** for _each_ Segment source that will be connected to the data lake
* The generated **EMR Cluster ID**
* The generated **IAM Role ARN**

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

The EMR cluster requires a route table attached to the subnet with an internet gateway. You can follow [this guide](https://aws.amazon.com/blogs/big-data/launching-and-running-an-amazon-emr-cluster-inside-a-vpc/) for guidance on creating and attaching a route table and internet gateway.

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

If all else fails, teardown and start over.

# Supported Terraform Versions

Terraform 0.11 or higher is supported.

NOTE: Release v0.2.0 onwards only Terraform 0.12 or higher is supported.

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
