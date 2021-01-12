Deploy EKS
==========

Prepare the VPC
---------------

Okay, so I just have to add some eks-vpc-tags to the VPC module in VPC-app, add the eks-cluster-name.

Huh, this guide says now to add the dns forwarder, which I think I already added because the outputs in the VPC app example already included it.

I also needed to set `tag_for_use_with_eks` to true! I already had `eks-cluster-name` in there.

Also eks_cluster_names is the new variable.

Services/eks-cluster
--------------------
First have to add the cluster-control-plane, then the workers.
Then have to do a bunch of setup stuff, and finally deploy.

For the control-plane. There's some discrepancy between the guide and what's in the terraform-aws-eks, so I'm just combining them, taking a while to do.

Packer template
---------------

I found eks-node-al2.json in the service catalog, so I'm going to build the AMI using that template instead of the template provided in the guide.

Looks like that AMI got built. I can see it in the AWS UI.

User Data Script
----------------
Now I'm not sure if I should use the script in the guide, which is more detailed, or the one in the aws-service-catalog. I think I'll use the guide one, because it has some stuff for CloudWatch.

Logging/metrics/alarms
---------------
If I want these to work, I have to already have a global sns topic set up. Huh. So why doesn't this guide help me figure that out?

Role-mapping
------------
This piece seems to be useful only if you have more users. I currently have a single IAM user who is an admin. But for the sake of following the guide, I'll go ahead and add role-mapping.
Oh, I see, by default the role-mappings are just {}, so it will work without any more users.

In order for this to work, it requires some "hackery" (as mentioned in the guide). I wonder if it matters which provider version I use for kubernetes. The guide says

```
provider "kubernetes" {
  version = "~> 1.6"
  ...
```

But the current version is 1.13.3.

And the AWS Service catalog uses this:

```
provider "kubernetes" {
  load_config_file       = false
  host                   = data.template_file.kubernetes_cluster_endpoint.rendered
  cluster_ca_certificate = base64decode(data.template_file.kubernetes_cluster_ca.rendered)
  token                  = data.aws_eks_cluster_auth.kubernetes_token.token
}
```

I don't know what to use here.


Configure access to control plane and worker nodes
---------------------------------------------------

In this section, I think I can safely skip everything except for giving worker nodes IAM permissions to talk to IAM, since I believe I added some stuff for ssh-grunt when setting up the security-baseline. But wait, I dont actually have any users in that group. So maybe I don't have to do anything for this section at the moment.

I think I'll attempt to deploy without doing this part, and then come back to this.

Deploy the EKS cluster
----------------------

I expect a lot of variable and output discrepancies before I can do this successfully.
Gah, that took forever. Getting all the variables updated and set correctly.
The eks cluster is now creating. I'm sure something will fail along the way still.

I know that the examples in the terraform-aws-eks repo work, so I can fallback to that if needed. I think we might want to update this guide to deploy Fargate eks clusters, using the example in that repo.

Gah, my packer build is in the wrong region. Rather than try to figure out how to fix that, I'm going to build a new image in the right region.

Had to work through a couple of other issues:
- actually need to make the EKS cluster publicly accessible for the first deploy, so that role-mappings can get set up. otherwise, you have to deploy it from within the VPC, and I don't feel like doing that. Also apparently that's a yak shave.
- I had to realize that endpoint_public_access_cidrs should be my public IP (whatismyip.com)

The deploy completed!
