Deploy the ref arch by yourself using modules
=============================================

Who
----
User: Subscriber to Gruntwork who doesn't want to buy our ref arch. They have access to
all our modules and the acme example repos.

Goal
----
Deploy a sample app along with its prerequisites (LZ, VPC, App Cluster, RDS, etc).

What's missing
--------------
One of the top things missing is an overview of what steps you need to take to deploy
enough of a reference architecture to get an app running on an app cluster.
Other missing things are added to the [PR](https://github.com/gruntwork-io/gruntwork-io.github.io/pull/387) for updating the documentation.

Steps
-----
Below is the list that I gathered from talking to folks at Gruntwork. It would be worth
mentioning these items in a guide, before diving into the landing zone guide.

1. [Deploy landing zone](1_deploy_landing_zone.md)
1. [Deploy a VPC](2_deploy_vpc.md)
1. [Deploy an app cluster in EKS](3_deploy_eks.md)
    - (either EKS or ECS). Can use ASG if you don't want an app cluster.
1. Deploy the sample app, frontend and backend.

Also implement:
1. Set up a way to pull secrets using the sample-app backend.
1. Set up a secret for the database password.
1. Make CRUD requests against the database.

Nice to haves:
1. Set up ci/cd pipeline.
1. Set up monitoring.
