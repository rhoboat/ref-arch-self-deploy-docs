Deploy VPC
==========

This all went pretty well, except that there is no guide for deploying a bastion host.

I will simply follow the bastion host examples in the service catalog.

Issue with terraform state from mgmt vpc
----------------------------------------
Psyche! It is not working right now. Hooking up the dns_to_mgmt_vpc module is not defined in the docs, so I have to figure it out.
The terraform_remote_state key path doesn't look right. It says to use region/mgmt/vpc/terraform.tfstate, but if you follow the guide,
I think it's region/prod/networking/vpc-mgmt/terraform.tfstate.

Agh, first of all, there was a typo, so it wasn't necessarily the bucket path. It might have just been the `.outputs.` piece was missing.
Oh, wait, it's saying the BUCKET doesn't exist, the key shouldn't matter. So I need to figure out why....
Okay, the bucket existing issue was a typo in my bucketname. I fixed it by setting it in common.hcl.

Now it turns out the bucket key was also wrong, geez.
Oh my god. There's a discrepancy between the recommended folder structure for mgmt stuff and our examples in aws-service-catalog.
aws-service-catalog has this structure:

----
infrastructure-live
  └ production
    └ us-east-2
      └ mgmt
        └ networking
          └ vpc-mgmt
            └ terragrunt.hcl
      └ prod
        └ networking
          └ vpc
            └ terragrunt.hcl
----

The guide has this structure:

----
infrastructure-live
  └ production
    └ us-east-2
      └ prod
        └ networking
          └ vpc-app
            └ terragrunt.hcl
          └ vpc-mgmt
            └ terragrunt.hcl
----

When deploying it, I got an error:
> Error: Error waiting for VPC Peering Connection to become available: Error waiting for VPC Peering Connection (pcx-0a366a07e11c91835) to become available: Failed due to incorrect VPC-ID, Account ID, or overlapping CIDR range

  - This was my mistake. I copied the original mgmt vpc terragrunt.hcl because the example had a lot of stuff that was off, and I didn't noticed that we're using a different CIDR range for app vpc.

I got an error again that some things were not there, so I chalked it up to a state mismatch again, and removed the state file from s3. I didn't realize that I also had to remove the entry in dynamodb for the state file! Hopefully now it will recreate everything. I also of course deleted the app vpc manually from the AWS UI.

Now I got an issue that I exceeded the EIP limit. So I went into clickops and removed the ones that didn't look associated to mgmt vpc nat gateways.

But it still exceeds the limit. I can't have both Mgmt and App VPCs set up. I need to request a limit increase.

Oh, well, just deleting all the stuff for mgmt vpc was enough to get my app vpc up. I wonder though if I messed something up. So I did request the limit increase anyway.

I'm going to hold off on deploying other App VPCs for dev and stage

Bastion Host
------------
Ugh. I don't feel like doing this. I want to just deploy the app already.
