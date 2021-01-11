Create an account in AWS, using root user: rho+test-account-1@gruntwork.io.
Set BREX card as billing information.
Sign up for free tier.

Add MFA to root user in root account.
root account id: `453687634329`
No IAM users in the root account; only the root user.

Go to My Organization, create Organization.
Create security account, rho+security-account-1@gruntwork.io

1. Landing Zone. Follow the actual walkthrough! Not just the design part.
- set up the root user and a new admin user properly
- create infrastructure-live/landingzone/account-baseline-root wrapper module
- create github repo for infrastructure-live
- set up testing with inner testing dir, backend.hcl and terraform.tfvars
- create aws-vault setup for rho-dive, in order to test the module
  - have to do this for the ROOT user, not the admin user!
- run `terraform init -backend-config=backend.hcl ../` and `terraform apply ../`
When running `terraform apply`, get this error:
```
Warning: Provider source not supported in Terraform v0.12

on .terraform/modules/root_baseline/modules/aws-config-multi-region/main.tf line 10, in terraform:
10:     aws = {
11:       source  = "hashicorp/aws"
            12:       version = ">= 2.58"
            13:     }

            A source was declared for provider aws. Terraform v0.12 does not support the
            provider source attribute. It will be ignored.

(and one more similar warning elsewhere)
```

So we need to update the example to use Terraform v0.13 and higher. However, this is just a warning, so we can still `terraform apply` and move past it.

- Fill out missing variables in testing/terraform.tfvars
- Rerun `aws-vault exec rho-dive -- terraform apply ../`
- If you already created accounts in your organization, you have to create all new accounts. You can do this by using new email addresses. But now you have duplicate accounts, for which you may not have the passwords. Without the passwords you can't sign into them and verify your info, which you need to do before you can delete them.
- Get error: Error: Creating Delivery Channel failed: InsufficientDeliveryPolicyException: Insufficient delivery policy to s3 bucket: rho-testing-refarchdeepdive-logs-config, unable to write to bucket, provided s3 key prefix is 'null', provided kms key is 'null'.
  - so I need to set up a s3 key prefix and kms key too?
  - So, not only does the walkthrough fail to mention that you need to create an s3 bucket before you run `terraform init`, but you also need to make sure the default settings of blocking all public access is turned OFF. Otherwise, terraform doesn't have rights to use it as the remote state backend.
- Meanwhile I removed the rho-admin user from IAM because I thought I messed up. I didn't realize I could import that user later. So I deleted the user and skipped the import step.
- I also incorrectly set up my aws-vault early on, before the step was mentioned in the docs, Because I needed it set up for testing the module, which is bad advice. I ended up setting up my aws-vault user to rho-admin rather than the root account, so when I deleted the rho-admin IAM user, I lost the aws-vault creds. It took me a while to set that up again, pointing to the root account. What a pain.
- Also I ended up not fully realizing when we shifted from adding code in infrastructure-modules vs infrastructure-live. So I ended up not realizing I had to create both repos to start. Why isn't that mentioned at the top? I shouldn't have to read through the entire "Production-grade design" section to learn that I needed two repos. That should have been mentioned at the start of the walkthrough.
So my setup now looks like this:
```
/Users/rhozen/Development/deep-dive/
▾ infrastructure-live/
  ▸ .git/
  ▾ root/_global/account-baseline/
    ▸ .terragrunt-cache/6nIPaZJnTT2fXDxddbSKhcAirwc/CZzO-tTdES2-ONF1vwyGGXiz2WI/
      backend.hcl
      terragrunt.hcl
▾ infrastructure-modules/
  ▸ .git/
  ▾ landingzone/account-baseline-root/
    ▾ testing/
        backend.hcl
        terraform.tfvars
      main.tf
      outputs.tf
      variables.tf
    .gitignore
  README.md
```
And I had to run `aws-vault exec rho-dive -- terragrunt init -backend-config=backend.hcl` and then `terragrunt apply`. These instructions are also not mentioned.
- Now I get this error when running apply:

Error: The role "arn:aws:iam::497155131447:role/OrganizationAccountAccessRole" cannot be assumed.

  There are a number of possible causes of this - the most common are:
    * The credentials used in order to assume the role are invalid
    * The credentials do not have appropriate permission to assume the role
    * The role ARN is not valid
  on .terraform/modules/root_baseline/modules/account-baseline-root/logs-account-resources.tf line 59, in provider "aws":
  59: provider "aws" {
  - to solve this, I am just going to bump up my email for the logs account and force it to create a new account. ARGH! That seems to get by the error.
- The apply just hung, didn't even show that it timed out, but it probably did. So now I'm running apply again, and I still get that same error. Which means I have to create yet another new logs account.
- Okay, step back. Looks like the issue is that I can't assume roles as the root account user. So I'll have to do the import steps.
  - I had an older version of terragrunt (0.23.x) so the command to run terragrunt aws-provider-patch failed. Got 0.26.7 and it worked.
  - Import worked!
  - Imported the organization and child accounts also. Resources are already managed by Terraform (the child accounts).
- Reran `terragrunt init -backend-config=backend.hcl` and `terragrunt apply`.
- Can you run `ulimit -n 1024` in another terminal session and will it still apply to the currently running process?
- Got errors:
Error: error deleting S3 Bucket (rho-testing-refarchdeepdive-logs-config): BucketNotEmpty: The bucket you tried to delete is not empty. You must delete all versions in the bucket.

status code: 409, request id: 6F9A8674CCF9A06A, host id: bDaUAXXiYGV/fpAjFYP8hx0fGNprduoZ0jsDHsi0KNLWSa+U1I6SOX4uJuV2dOLKp2t29sYTSTM=

Error: Creating Delivery Channel failed: InsufficientDeliveryPolicyException: Insufficient delivery policy to s3 bucket: rho-testing-refarchdeepdive-logs-config, unable to write to bucket, provided s3 key prefix is 'null', provided kms key is 'null'.

on .terraform/modules/root_baseline/modules/aws-config/main.tf line 223, in resource "aws_config_delivery_channel" "config_delivery_channel":

223: resource "aws_config_delivery_channel" "config_delivery_channel" {

Error: Error creating CloudTrail: InvalidCloudWatchLogsLogGroupArnException: Check the log group ARN: CloudTrail can't validate it.

on .terraform/modules/root_baseline/modules/cloudtrail/main.tf line 143, in resource "aws_cloudtrail" "cloudtrail":

143: resource "aws_cloudtrail" "cloudtrail" {
  - It looks like the error deleting S3 bucket is a known issue. In that case we should document it as something you should expect to see.
  - Why am I seeing the InsufficientDeliveryPolicyException? The module created all these resources, not me.
  - Rerunning apply several times shows the same errors. This is a huge barrier.
  - To solve this, I renamed the name_prefix back to the older name, which I used when mistakenly trying to test the module, and for the error creating CloudTrail, I found that module-security 0.35.x and above supports aws provider version 3.x, so all I have to do is bump the provider version locally to 3.x and it should work. Rerun init and apply.
  - But I also have to make sure to update the tags in infrastructure-modules so that infrastructure-live pulls those changes!
- Done, seems to work. I didn't commit the changes yet.
- Going through all the accounts to reset the root account passwords, to the same as rho-admin.
- Apply security baseline to the logs account
  - Involved me looking up exactly how to set up the remote state block and where it should go. Updated the docs to mention that.
  - Wow, that worked on the first try.
- Apply security baseline to the security account
  - Found some typos, and also had to overhaul how I'm setting up the state buckets. Included now a common.hcl and region/account.hcls.
  - Found more typos with group names
- Apply app baseline to the dev, stage, prod, and shared-services accounts
