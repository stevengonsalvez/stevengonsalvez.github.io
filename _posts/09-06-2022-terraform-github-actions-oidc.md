# Using OIDC for terraform-azure with github actions for continous infrastructure.


So what was the approach prior to OIDC in terraform or availability of OIDC in github actions. 

The ways to do CICD for terraform infrastructure involved 
- using a managed identity, which mean't that you needed some "infrastructure" component in azure (pre-existing) to have a managed identity, to deploy and manage continuously other infrastructure. So that would involve
	- Some pre-existing (VM or containers (kube or other))
	- For VMs/Containers, those could be used as runners for the terraform (from any CI tooling) or some script/orchestrator to execute (on pre-existing VMs. Eg: ansible or even make or we could overengineer some terraform to do terraform)

- using a service principal and a relatively "static" password/certificate. The credentials will need to be stored in a "vault" or in the case of github actions (github secrets), and present that as part of the job.


Both these approaches have its frailties and complexities. Either it involves complicated orchestration, more footprint management than is needed and more code/configuration than is intended. The latter, well - straightforward is either long-lived credentials, or some over-engineered secret management capability which will always have attack vectors - due to the considerably vulnerable nature of a static generated key


Federated OIDC solves both these problems, making it secure and much less fragile. 
(eg: shortlived tokens, granular access, no additional wrap or orchestration.)
If OIDC is new, start with  [these illustrations](https://developer.okta.com/blog/2019/10/21/illustrated-guide-to-oauth-and-oidc) 

A new solution of using passwordless service principal with OIDC to use short lived tokens with "sort of" federated identity between cloud provider identity and github actions (as an identity provider)


In the case of Github Actions, Github is used as an federated identity provider  with a cloud identity provider (for azure, AD). The identity object itself is facets of the workflow (environment , pullrequest , branch , tag)

A TLDR view of how federated workload identity works between github action and azure. 
![github-oidc](https://user-images.githubusercontent.com/9320602/172952577-9554fe38-5b5a-4a8a-a580-77a2997ee081.png)


## How it works. 

So there are three items in here. 

- The Service Principal setup itself (the service principal that is used to run the terraform)
- Terraform configuration for OIDC
- Github Actions and Github Secrets.


### The service principal

- The service principal will need to have the relevant role's (that is requried) eg: contributor to provision, as foremost.
- Need to create an azuread federated credential on the service principal, with the subject identifier of whatever is relevant for the pipeline (eg: pullrequest, branch, environment or tag)

