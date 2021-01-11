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
