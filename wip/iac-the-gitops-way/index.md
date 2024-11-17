# IaC the GitOps way

## Need

Our platform users needs to be able to deploy entire environments (network, kubernetes clusters and other resources)
without having to know the underlying mechanic. Ideally, when a developer needs a fresh environment to accomplish their
project, they should be able to create it automatically in a few step from a web interface or a configuration file in a
Git repository.

## Challenges

This highly automated approach raises some challenges :

- How to reliably manage the infrastructure lifecycle ?
- How to protect the infrastructure from being accidentally deleted ? (assuming critical environments are deployed
  with this method as well)
- How to abstract the underlying infrastructure as code projects so the entire system can be used accross multiple
  applications / perimeters ?
- How to communicate with the user on the stage of the deployment and to notify them than their temporary environment is
  too old regarding the policies ?
- How to implement policies to avoid deploying production workloads to a non-critical environment or account ?
- How to manage permissions on these automatically created environments ?
- Should the environment be automatically destroyable ?
- How to create the documentation for this deployment ?
- How to monitor these environment in flexible way ?

## Initial though

At first, I thought to implement a pipeline triggered on different configuration Git repositories. By cloning Terraform
modules repositories, it can mix everything and deploy the infrastructure after checking the compliance
with a policies repositories. To handle the lifecycle of the infrastructure, another pipeline should run now and
then to detect expired infrastructure (if TTL was implemented). I should also destroy infrastructure when the
configuration file has been deleted. But...

[//]: # (@formatter:off)
/// admonition | It's the canonical role of a Kubernetes Controller
    type: tip
///
[//]: # (@formatter:on)

So I moved on to see if a Terraform Kubernetes Controller exists. Unfortunatly, the official Terraform Cloud Controller
forces us to use the Terraform Cloud. Which is a paid service... (1)
{ .annotate }

1. Not a fan of what Terraform is becoming due to Hashicorp philosophy...

Fortunately for us, there is a competitor : Pulumi. Which is open source, free and seems to attached of the
open source principles (Their Cloud platform service is paid, but it's legit). Pulumi has several benefits over
Terraform:

* It's IaC in real programming language (Java, C#, Typescript, Python and Go)
    * It does not re-invent the wheel: package management, language...
    * It provides the power of IaC AND the power of a real programming language with thousands of libraries and the
      flexible logic.
* The Kubernetes Operator is free and almost production ready[^1] and does not require an account on the Pulumi Cloud
* Pulumi Cloud can be self-hosted (still paid) to benefit from a graphical interface to manage the stacks.
    * The pricing model makes more sens than many out there: you don't pay for the number of users but for the number of
      resources Pulumi Cloud is managing.

[^1]: At the time I wrote this article: https://github.com/pulumi/pulumi-kubernetes-operator/tree/v2.0.0-beta.2

Since we are talking about GitOps, it makes sens to use Flux or ArgoCD. I'm more familiar with ArgoCD so I'll go with
it. For the Git repositories, any of the major ones will do. The choice here is no relevant because it's a theoretical
article.

For the enforcement of policies, Pulumi provide with a Policies as Code
solution: [Cross guard](https://www.pulumi.com/docs/iac/packages-and-automation/crossguard/). The policies can be stored
in a different Git Repository managed by the platform admin (not accessible by the user)

Finally, the TTL management. Pulumi Cloud provides with a tool which handles this for
us: [Time-to-live stacks](https://www.pulumi.com/docs/pulumi-cloud/deployments/ttl/). But I'll try to keep things
free[^2] so we will whip up a never restarting pod that ends when the TTL expires with an argocd
event that notifies the users when the pod ends.

[^2]: I'm not rich enough to ay for personal projects

# Tech choices

* ArgoCD to continuously deliver the new configurations.
* Pulumi as the main IaC engine.
* Pulumi Kubernetes Operator to automate the deployment of the IaC configurations.
* Pulumi Cross guard for the policies.
* A pod that stops when TTL ends and triggers an ArgoCD event.
* Helm to wrap up the Kubernetes resources.
* An AWS S3 Bucket to store the IaC states