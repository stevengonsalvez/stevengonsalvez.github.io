---
title: Github OIDC with terraform and Azure
published: false
description: Securing GitHub Actions with OIDC to create continuous infrastructure on Azure
tags: 'azure, github oidc, terraform, oidc'
cover_image: https://user-images.githubusercontent.com/9320602/177221304-5146ca58-f08f-4278-9afc-6d49d2e8e6de.png
canonical_url: null
---
# Securing GitHub Actions with OIDC to create continuous infrastructure on Azure

Continuous deployment workflows can deploy and manage lifecycle of cloud infrastructure on Azure, AWS, GCP etc. To do this, workflows would need to authenticate itself against the cloud provider using credentials that can be accessed from GitHub Secrets or an external vault appropriately. GitHub OIDC with cloud providers is now available to use, without the complication of credential management.

## Using OIDC for terraform-azure with GitHub actions for continuous infrastructure.


So what was the approach prior to OIDC in terraform or availability of OIDC in GitHub actions. 

The ways to do CI/CD for terraform infrastructure involved 
- Using a managed identity, which mean't that you needed some "infrastructure" component in azure (pre-existing) to have a managed identity, to deploy and manage continuously other infrastructure. So that would involve
	- Some pre-existing (VM or containers (Kube or other))
	- For VMs/Containers, those could be used as runners for the terraform setup (from any CI tooling) or some script/orchestrator to execute (on pre-existing VMs. e.g.: ansible or even make, or we could overengineer some terraform to do terraform)

- Using a service principal and a relatively "static" password/certificate. The credentials will need to be stored in a "vault" or in the case of GitHub actions (GitHub secrets), and present that as part of the job.


Both these approaches have its frailties and complexities. Either it involves complicated orchestration, more footprint management than is needed and more code/configuration than is intended. The latter, well - straightforward is either long-lived credentials, or some overengineered secret management capability which will always have attack vectors - due to the considerably vulnerable nature of a static generated key


Federated OIDC solves both these problems, making it secure and much less fragile. 
(e.g.: short-lived tokens, granular access, no additional wrap or orchestration.)
If you are new to OIDC, start with [these illustrations](https://developer.okta.com/blog/2019/10/21/illustrated-guide-to-oauth-and-oidc) 

A new solution of using passwordless service principal with OIDC to use short-lived tokens with "sort of" federated identity between cloud provider identity and GitHub actions (as an identity provider)

In the case of GitHub Actions, GitHub is used as a federated identity provider with a cloud identity provider (for azure, AD). The identity object itself is facets of the workflow (environment, pull-request, branch, tag)

> A TLDR view of how federated workload identity works between GitHub action and azure. 

![GitHub-oidc](https://user-images.GitHubusercontent.com/9320602/172952577-9554fe38-5b5a-4a8a-a580-77a2997ee081.png)


## How it works. 

To demonstrate the working, will be using Terraform to provision infrastructure on Azure.

So there are three items in here. 

- The Service Principal setup itself (the service principal that is used to run terraform to provision)
- Terraform configuration for OIDC
- GitHub Actions and GitHub Secrets.


### The service principal

- The service principal will need to have the relevant role's (that is required) e.g.: contributor to provision, as foremost.
- Need to create an azure ad federated credential on the service principal, with the subject identifier(s) of whatever is relevant for the pipeline (e.g.: pull-request, branch, environment or tag)

**Creating a service principal with terraform and assigning contributor access to the subscription**

</br>

> The API permissions needed to create this credential (either with manually as a user or if using another service principal to create this service principal)
> - Application.readwrite.All
> - User.read.All (This is required to look up user's object_id to assign it as owner of Service Principal)
> - Group.read.All (This is required to look up Group's object_id to assign it as owner of Service Principal)

```HCL
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.9.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = ">= 2.22.0"
    }
  }
}

provider "azurerm" {
  features {}
}


data "azurerm_subscription" "this" {
}

locals {
  scope    = data.azurerm_subscription.this.id
  app_name = "tf-oidc-test-sample"
}

data "azuread_client_config" "current" {}

data "azuread_user" "this" {
  user_principal_name = var.owner
}

resource "azuread_application" "this" {
  display_name = local.app_name
  web {
    implicit_grant {
      access_token_issuance_enabled = true
    }
  }
  owners = [data.azuread_client_config.current.object_id, data.azuread_user.this.object_id]
}

resource "azuread_service_principal" "this" {
  application_id               = azuread_application.this.application_id
  app_role_assignment_required = false
  lifecycle {
    prevent_destroy = false
  }
}

resource "azuread_application_password" "this" {
  application_object_id = azuread_application.this.id
  display_name          = "tf-credentials"
  end_date              = "2099-01-01T01:02:03Z"
}


resource "azurerm_role_assignment" "sub-contributor" {
  scope                = local.scope
  role_definition_name = "Contributor"
  principal_id         = azuread_service_principal.this.id
  // If new SP there  may be replciation lag this disables validation
  skip_service_principal_aad_check = true
  lifecycle {
    ignore_changes = [
      scope,
    ]
  }
}
```


### Create a federated credential on the Service principal

</br>

> The API permissions needed to create this credential (either with manually as a user or if using another service principal to create this service principal)
> - Application.readwrite.ownedby (or all)



```HCL
resource "azuread_application_federated_identity_credential" "main" {
  application_object_id = azuread_application.this.id
  display_name          = "az-oidc-branch-main"
  description           = "deployments for repository cloud-cicd-exploration"
  audiences             = ["api://AzureADTokenExchange"]
  issuer                = "https://token.actions.githubusercontent.com"
  subject               = "repo:stevengonsalvez/cloud-cicd-exploration:ref:refs/heads/main"
}

resource "azuread_application_federated_identity_credential" "pr" {
  application_object_id = azuread_application.this.id
  display_name          = "az-oidc-pr"
  description           = "deployments for repository cloud-cicd-exploration"
  audiences             = ["api://AzureADTokenExchange"]
  issuer                = "https://token.actions.githubusercontent.com"
  subject               = "repo:stevengonsalvez/cloud-cicd-exploration:pull_request"
}

resource "azuread_application_federated_identity_credential" "env-prod" {
  application_object_id = azuread_application.this.id
  display_name          = "az-oidc-env-prod"
  description           = "deployments for repository cloud-cicd-exploration"
  audiences             = ["api://AzureADTokenExchange"]
  issuer                = "https://token.actions.githubusercontent.com"
  subject               = "repo:stevengonsalvez/cloud-cicd-exploration::environment:production"
}
```

The above example creates one federated credential for each of
- environment: environment is `production` on the job.
- pull request: Any job executing on the pull request event
- main branch: Any job executing on a `push` to the main branch

The order of precedence is as above. e.g.:

When used in GitHub actions, GitHub's workflows create a JWT with relevant claims that are passed back to the cloud identity provider to be validated against the assertions(subject identifier) set up in the Federated cred

For the following workflow, although the trigger event is a pull-request, the claims in the JWT sent back from the job `plan-rg` will be the environment `Nonprod`. Therefore, the service principal will need to have a federated credential that contains the subject identifier as such `repo:<owner>/<repository>:environment:Nonprod`

>Note: Also this is case-sensitive

The JWT will look something like the below (refer to `sub` claim in jwt) - when executed on the [repository](https://github.com/stevengonsalvez/cloud-cicd-exploration)

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "x5t": "example-thumbprint",
  "kid": "example-key-id"
}
{
  "jti": "some-id",
  "sub": "repo:stevengonsalvez/cloud-cicd-exploration:environment:Nonprod",
  "environment": "prod",
  "aud": "https://github.com/stevengonsalvez",
  "ref": "refs/heads/branch",
  "sha": "example-sha",
  ... bunch of other stuff
}
```

```yaml
name: az-oidc-test

on:
  pull_request:
    branches:
      - master

permissions:
  id-token: write
  contents: read
  pull-requests: write
  issues: write
  statuses: write


# subject on oidc : pullrequest or environment:
# if environment specified on job (pull request) does not work
jobs:
  plan-rg:
    runs-on: ubuntu-latest
    name: plan rg
    environment:
      name: Nonprod
    env:
      ARM_SUBSCRIPTION_ID: ${{ secrets.SUBSCRIPTION_ID }}
      ARM_CLIENT_ID: ${{ secrets.SP_CLIENT_ID }}
      ARM_TENANT_ID: ${{ secrets.TENANT_ID }}
      LAYER_NAME: resource-group
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: debug
        run: |
          echo "$GITHUB_CONTEXT"
          env
          echo ${ACTIONS_ID_TOKEN_REQUEST_URL}
          echo ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}
```


### Terraform configuration

The terraform configuration is straightfoward. Terraform providers inherently use these helpers which now has integrated with oidc. See [here](https://github.com/hashicorp/go-azure-helpers/pull/115/files).

The detail of configuring azurerm provider in terraform to use oidc is [here](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_oidc). 

The short TLDR version of using OIDC with GitHub actions is simple. 

- Either in provider section of terraform, specify `use_oidc` as below. Refer [example](https://github.com/stevengonsalvez/cloud-cicd-exploration/blob/master/terraform/resource-group/main.tf#L12)

```terraform
provider "azurerm" {
  use_oidc = true
  features {}
}
```

- Or set the environment variable `ARM_USE_OIDC=true`

For GitHub Actions there is no need to specify the ID_URL and ID_token, as that seems to be integrated into the [azurerm provider](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/internal/provider/provider.go#L179) (Although, it is strange the decision to couple terraform provider with a particular CI/CD tool)

> Note: If using az cli outside the context of terraform as a separate step in GitHub actions job or as a local-exec in terraform, that would need to be authenticated using OIDC with the [azure/login](https://github.com/Azure/login#github-action-for-azure-login) action


### GitHub Actions configuration

The only setting needed for GitHub actions to be able to authenticate with the cloud provider is `permissions` either at job or workflow.

```yaml
permissions:
  id-token: write
```

*For a working terraform/actions example:*

- Service principal setup: [here](https://github.com/stevengonsalvez/cloud-cicd-exploration/tree/master/terraform/service-principal-oidc)
- Provisioning a [resource group](https://github.com/stevengonsalvez/cloud-cicd-exploration/blob/master/terraform/resource-group/main.tf) through a GitHub actions workflow - [here](https://github.com/stevengonsalvez/cloud-cicd-exploration/blob/master/.github/workflows/az-oidc-test.yml)

</br></br>

## How to GitOps the whole setup

</br>

For any organisation with lots of apps and repositories, each repository will need to have a service principal with access to an appropriate subscription (whatever subscription strategy is employed)

>Note: It is strongly advised not to share the same service primary as part of least-privilege.

Consider 50+ repositories, each with its own terraform assembly of modules (or a dependency injection style execution - covered in another post), that require a service principal to install and administer azure cloud infrastructure.

A manual approach for creating service principals and managing lifecycle (revoking, permissions etc.), will significantly reduce flow and the goal would be to automate the mechanism. Treat it like a vending machine, issuing the appropriate access when provided with the appropriate identity via a GitOps setup (completely code managed and requests/changes via a pull request, automated via GitHub's actions workflows).

The following is a detail of a GitOps setup requesting service principals for a repository

![image](https://user-images.githubusercontent.com/9320602/177196497-9ef5917d-de95-40ee-96a4-3fd7d99d6366.png)

A mock request pull request to the `service principal provisioner` would be of form (in a request_1.tfvar)

```hcl
owners = { users=[user.name@organisation.com], groups=[group@organisation.com] } # owners of the service principal
app_name = test-oidc-demo #name of the service principal
subscriptions = [ subscription1, subscription2 ] #subscriptions that this SP requires access to
federated_credentials_env = [ nonprod, prod, stage ] #environments that are used in github workflow jobs.
```

A working example [here](https://github.com/stevengonsalvez/cloud-cicd-exploration/tree/master/terraform/az-vending-service-principal)

This would generate service principals, that can be injected into a vault or into a GitHub secrets. Only the application_id of the service principal is needed. No passwords are generated. So this can be directly injected into configuration if need be.

>e.g.: injecting into GitHub secrets

```
resource "github_actions_environment_secret" "client_id" {
  repository  = "repo_name"
  environment = "environment_name"
  secret_name     = "ARM_CLIENT_ID"
  plaintext_value = module.service_principal.id # or whatever is the reference appropriately.
}
```
> API Permissions needed for the Azure Vending machine Service Principal 
> 	- Application.readwrite.all
>   - User.read.All


</br></br>

**There is one issue with the above setup to iron out**

- The Azure Vending machine is now having environment related configuration for the target application, which makes it coupled with its lifecycle (e.g.: environment name change in workflows from stage to pre-prod, will need a change from the vending machine to re-issue credentials)

One solution would be

- Make the target application Service principal that is created an owner for itself
- Assign `Application.readwrite.ownedby` access to the service principal
- The terraform portion of generating federated credentials can then be generated as part of bootstrap in the pipeline of that repository.

Consequently, all environment lifecycle related parts of service principal is maintained locally with the application configuration - as detailed. 

![image](https://user-images.githubusercontent.com/9320602/177217647-a4f26235-6e44-4d17-b905-2f4f13de987e.png)

>Permissions required for the Azure Vending Machine service principal
> 	- Application.readwrite.All (API permissions)
>   - User.read.All (API permissions)
>   - `Global Administrator or Privileged Role Administrator` to grant API permissions to service principals

## Appendix

Working examples in [CICD-exploration repository](https://github.com/stevengonsalvez/cloud-cicd-exploration)
