---
copyright:
  years: 2018
lastupdated: "2018-06-05"

---

{:java: #java .ph data-hd-programlang='java'}
{:swift: #swift .ph data-hd-programlang='swift'}
{:ios: #ios data-hd-operatingsystem="ios"}
{:android: #android data-hd-operatingsystem="android"}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}

# Plan, create and update deployment environments

Multiple deployment environments are common when building a solution. They reflect the lifecycle of a project from development to production. This tutorial introduces tools like the {{site.data.keyword.Bluemix_notm}} CLI and [Terraform](https://www.terraform.io/) to automate the creation and maintenance of these deployment environments.
{:shortdesc}

Developers do not like to write the same thing twice. The [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle is one example of this. Similarly they don't like having to go through tons of clicks in a user interface to setup an environment. Consequently shell scripts have been long used by system administrators and developers to automate repetitive, error-prone and uninteresting tasks.

Infrastructure as a Service (IaaS), Platform as a Service (PaaS), Container as a Service (CaaS), Functions as a Service (FaaS) have given developers high level of abstraction and it became easier to acquire resources like bare metal servers, managed databases, virtual machines, Kubernetes clusters, etc. But once you have provisioned these resources, you need to connect them together, to configure user access, to update the configuration over time, etc. Being able to automate all these steps and to repeat the installation, configuration under different environments is a must-have these days.

Multiple environments are pretty common in a project to support the different phases of the development cycle with slight differences between the environments like capacity, networking, credentials, log verbosity. In [this other tutorial](./users-teams-applications.html), we've introduced best practices to organize users, teams and applications and a sample scenario. The sample scenario considers three environments, *Development*, *Testing* and *Production*. How to automate the creation of these environments? What tools could be used?

## Objectives
{: #objectives}

* Define a set of environments to deploy
* Write scripts using the {{site.data.keyword.Bluemix_notm}} CLI and [Terraform](https://www.terraform.io/) to automate the deployment of these environments
* Deploy these environments in your account

## Services used
{: #services}

This tutorial uses the following products:
* [{{site.data.keyword.Bluemix_notm}} provider for Terraform](https://ibm-cloud.github.io/tf-ibm-docs/index.html)
* [{{site.data.keyword.containershort_notm}}](https://console.bluemix.net/containers-kubernetes/catalog/cluster)
* [Identity and Access Management](https://console.bluemix.net/iam/#/users)
* [{{site.data.keyword.Bluemix_notm}} command line interface - the `ibmcloud` CLI](https://console.bluemix.net/docs/cli/index.html)
* [HashiCorp Terraform](https://www.terraform.io/)

This tutorial may incur costs. Use the [Pricing Calculator](https://console.bluemix.net/pricing/) to generate a cost estimate based on your projected usage.

## Architecture
{: #architecture}

<p style="text-align: center;">

![](./images/solution26-plan-create-update-deployments/architecture.png)
</p>

1. A set of Terraform files are created to describe the target infrastructure as code.
1. An operator uses `terraform apply` to provision the environments.
1. Shell scripts are written to complete the configuration of the environments.
1. The operator runs the scripts against the environments
1. The environments are fully configured, ready to be used.

## Overview of the available tools
{: #tools}

The first tool to interact with {{site.data.keyword.Bluemix_notm}} and to create repeatable deployments is the [{{site.data.keyword.Bluemix_notm}} command line interface - the `ibmcloud` CLI](https://console.bluemix.net/docs/cli/index.html). With `ibmcloud` and its plugins, you can automate the creation and configuration of your cloud resources. {{site.data.keyword.virtualmachinesshort}}, Kubernetes clusters, {{site.data.keyword.openwhisk_short}}, Cloud Foundry apps and services, you can provision all of them from the command line.

Another tool introduced in [this tutorial](./infrastructure-as-code-terraform.html) is [Terraform](https://www.terraform.io/) by HashiCorp. Quoting HashiCorp, *Terraform enables you to safely and predictably create, change, and improve infrastructure. It is an open source tool that codifies APIs into declarative configuration files that can be shared amongst team members, treated as code, edited, reviewed, and versioned.* It is infrastructure as code. You write down what your infrastructure should look like and Terraform will create, update, remove cloud resources as needed.

To support a multi-cloud approach, Terraform works with providers. A provider is responsible for understanding API interactions and exposing resources. {{site.data.keyword.Bluemix_notm}} has [its provider for Terraform](https://github.com/IBM-Cloud/terraform-provider-ibm) enabling users of {{site.data.keyword.Bluemix_notm}} to manage resources with Terraform. Although Terraform is categorized as infrastructure as code, it is not limited to Infrastructure-As-A-Service resources. The {{site.data.keyword.Bluemix_notm}} Provider for Terraform supports IaaS (bare metal, virtual machine, network services, etc.), CaaS ({{site.data.keyword.containershort_notm}} and Kubernetes clusters), PaaS (Cloud Foundry and services) and FaaS ({{site.data.keyword.openwhisk_short}}) resources.

## Write scripts to automate the deployment
{: #scripts}

As you start describing your infrastructure-as-code, it is critical to treat files you create as regular code, thus storing them in a source control management system. Overtime this will bring good properties such as using the source control review workflow to validate changes before applying them, adding a continuous integration pipeline to automatically deploy infrastructure changes.

[This Git repository](https://github.com/IBM-Cloud/multiple-environments-as-code) has all the configuration files needed to setup the environments defined earlier. You can clone the repository to follow the next sections detailing the content of the files.

   ```sh
   git clone https://github.com/IBM-Cloud/multiple-environments-as-code
   ```

The repository is structured as follow:

| Directory | Description |
| ----------------- | ----------- |
| [terraform](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/terraform) | Home for the Terraform files |
| [terraform/global](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/terraform/global) | Terraform files to provision resources common to the three environments |
| [terraform/per-environment](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/terraform/per-environment) | Terraform files specific to a given environment |
| [iam](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/iam) | Scripts to configure user permissions |

### Heavy lifting with Terraform

The *Development*, *Testing* and *Production* environments pretty much look the same.

<p style="text-align: center;">
  <img title="" src="./images/solution26-plan-create-update-deployments/one-environment.png" style="height: 400px;" />
</p>

They share a common organization and environment-specific resources. They will differ by the allocated capacity and the access rights. The terraform files reflect this with a ***global*** configuration to provision the Cloud Foundry organization and a ***per-environment*** configuration, using Terraform workspaces, to provision the environment-specific resources:

![](./images/solution26-plan-create-update-deployments/terraform-workspaces.png)

### Global Configuration

All environments share a common Cloud Foundry organization and each environment has its own space.

Under the [terraform/global](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/terraform/global) directory, you find the Terraform scripts to provision this organization. [main.tf](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/global/main.tf) contains the definition for the organization:

   ```sh
   # create a new organization for the project
   resource "ibm_org" "organization" {
     name             = "${var.org_name}"
     managers         = "${var.org_managers}"
     users            = "${var.org_users}"
     auditors         = "${var.org_auditors}"
     billing_managers = "${var.org_billing_managers}"
   }
   ```

In this resource, all properties are configured through variables. In the next sections, you will learn how to set these variables.

To fully deploy the environments, you will use a mix of Terraform and the {{site.data.keyword.Bluemix_notm}} CLI. Shell scripts written with the CLI may need to reference this organization or the account by name or ID. The *global* directory also includes [outputs.tf](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/global/outputs.tf) which will produce a file containing this information as keys/values suitable to be reused in scripting:

   ```sh
   # generate a property file suitable for shell scripts with useful variables relating to the environment
   resource "local_file" "output" {
     content = <<EOF
ACCOUNT_GUID=${data.ibm_account.account.id}
ORG_GUID=${ibm_org.organization.id}
ORG_NAME=${var.org_name}
EOF

     filename = "../outputs/global.env"
   }
   ```

### Individual Environments

There are different approaches to manage multiple environments with Terraform. You could duplicate the Terraform files under separate directories, one directory per environment. With [Terraform modules](https://www.terraform.io/docs/modules/index.html) you could factor common configuration as a group and reuse modules across environments - reducing the code duplication. Separate directories mean you can evolve the *development* environment to test changes and then propagate the changes to other environments. It is common in this case to also have the Terraform *modules* in their own source code repository so that you can reference a specific version of a module in your environment files.

Given the environments are rather simple and similar, you are going to use another Terraform concept called [workspaces](https://www.terraform.io/docs/state/workspaces.html#best-practices). Workspaces allow to use the same terraform files (.tf) with different environments. In the example, *development*, *testing* and *production* are workspaces. They will use the same Terraform definitions but with different configuration variables (different names, different capacities).

Each environment requires:
* a dedicated Cloud Foundry space
* a Kubernetes cluster
* a database
* a file storage

The Cloud Foundry space is linked to the organization created in the previous step. The environment Terraform files need to reference this organization. This is where [Terraform remote state](https://www.terraform.io/docs/state/remote.html) will help. It allows to reference an existing Terraform state in read-only mode. This is a very useful construct to split your Terraform configuration in smaller pieces leaving the responsibility of individual pieces to different teams. [backend.tf](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/per-environment/backend.tf) contains the definition of the *global* remote state used to find the organization created earlier:

   ```sh
   data "terraform_remote_state" "global" {
      backend = "local"

      config {
        path = "${path.module}/../global/terraform.tfstate"
      }
   }
   ```

Once you can reference the organization, it is straightforward to create a space within this organization. [main.tf](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/per-environment/main.tf) contains the definition of the resources for the environment.

   ```sh
   # a Cloud Foundry space per environment
   resource "ibm_space" "space" {
     name       = "${var.environment_name}"
     org        = "${data.terraform_remote_state.global.org_name}"
     managers   = "${var.space_managers}"
     auditors   = "${var.space_auditors}"
     developers = "${var.space_developers}"
   }
   ```

Notice how the organization name is referenced from the *global* remote state. The other properties are taken from configuration variables.

Next comes the Kubernetes cluster. The {{site.data.keyword.Bluemix_notm}} provider has a Terraform resource to represent a cluster:

   ```sh
   # a cluster
   resource "ibm_container_cluster" "cluster" {
     name            = "${var.environment_name}-cluster"
     datacenter      = "${var.cluster_datacenter}"
     org_guid        = "${data.terraform_remote_state.global.org_guid}"
     space_guid      = "${ibm_space.space.id}"
     account_guid    = "${data.terraform_remote_state.global.account_guid}"
     machine_type    = "${var.cluster_machine_type}"
     worker_num      = "${var.cluster_worker_num}"
     public_vlan_id  = "${var.cluster_public_vlan_id}"
     private_vlan_id = "${var.cluster_private_vlan_id}"
   }
   ```

Again most of the properties will be initialized from configuration variables. You can adjust the datacenter, the number of workers, the type of workers.

Cloud Foundry services can be provisioned and a Kubernetes binding (secret) can be added to retrieve the service credentials from your applications:

   ```sh
   # a database
   resource "ibm_service_instance" "database" {
     name       = "database"
     space_guid = "${ibm_space.space.id}"
     service    = "cloudantNoSQLDB"
     plan       = "Lite"
   }

   # bind the database service to the cluster
   resource "ibm_container_bind_service" "bind_database" {
    cluster_name_id             = "${ibm_container_cluster.cluster.id}"
    service_instance_space_guid = "${ibm_space.space.id}"
    service_instance_name_id    = "${ibm_service_instance.database.id}"
    namespace_id                = "default"
    account_guid                = "${data.terraform_remote_state.global.account_guid}"
    org_guid                    = "${data.terraform_remote_state.global.org_guid}"
    space_guid                  = "${ibm_space.space.id}"
   }
   ```

## Deploy this environment in your account

### Install {{site.data.keyword.Bluemix_notm}} CLI

1. Follow [these instructions](https://console.bluemix.net/docs/cli/reference/bluemix_cli/download_cli.html#download_install) to install the CLI
1. Validate the installation by running:
   ```sh
   ibmcloud
   ```
   {: codeblock}

### Install Terraform and the {{site.data.keyword.Bluemix_notm}} provider for Terraform

1. [Download and install Terraform for your system.](https://www.terraform.io/intro/getting-started/install.html)
1. [Download the Terraform binary for the {{site.data.keyword.Bluemix_notm}} provider.](https://github.com/IBM-Cloud/terraform-provider-ibm/releases)
   To setup Terraform with {{site.data.keyword.Bluemix_notm}} provider, refer to this [link](./infrastructure-as-code-terraform.html#setup)
   {:tip}
1. Create a `.terraformrc` file in your home directory that points to the Terraform binary. In the following example, `/opt/provider/terraform-provider-ibm` is the route to the directory.
   ```sh
   # ~/.terraformrc
   providers {
     ibm = "/opt/provider/terraform-provider-ibm"
   }
   ```
   {: codeblock}

### Get the code

If you have not done it yet, clone the tutorial repository:

   ```sh
   git clone https://github.com/IBM-Cloud/multiple-environments-as-code
   ```
   {: codeblock}

### Set Platform API key

1. If you don't already have one, obtain a [Platform API key](https://console.bluemix.net/iam/#/apikeys) and save the API key for future reference.
   > If in later steps you plan on creating a new Cloud Foundry organization to host the deployment environments, make sure you are the owner of the account.
1. Copy [terraform/credentials.tfvars.tmpl](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/credentials.tfvars.tmpl) to *terraform/credentials.tfvars* by running the below command
   ```sh
   cp terraform/credentials.tfvars.tmpl terraform/credentials.tfvars
   ```
   {: codeblock}
1. Edit `terraform/credentials.tfvars` and set the value for `ibmcloud_api_key` to the Platform API key you obtained.

### Create or reuse a Cloud Foundry organization

You can choose either to create a new organization or to reuse (import) an existing one. To create the parent organization of the three deployment environments, **you need to be the account owner**.

#### To create a new organization

1. Change to the `terraform/global` directory
1. Copy [global.tfvars.tmpl](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/global/global.tfvars.tmpl) to `global.tfvars`
   ```sh
   cp global.tfvars.tmpl global.tfvars
   ```
   {: codeblock}
1. Edit `global.tfvars`
   1. Set **org_name** to the organization name to create
   1. Set **org_managers** to a list of user IDs you want to grant the *Manager* role in the org - the user creating the org is automatically a manager and should not be added to the list
   1. Set **org_users** to a list of all users you want to invite into the org - users need to be added there if you want to configure their access in further steps

   ```sh
   org_name = "a-new-organization"
   org_managers = [ "user1@domain.com", "another-user@anotherdomain.com" ]
   org_users = [ "user1@domain.com", "another-user@anotherdomain.com", "more-user@domain.com" ]
   ```
   {: codeblock}
1. Initialize Terraform from the `terraform/global` folder
   ```sh
   terraform init
   ```
   {: codeblock}
1. Look at the Terraform plan
   ```sh
   terraform plan -var-file=../credentials.tfvars -var-file=global.tfvars
   ```
   {: codeblock}
1. Apply the changes
   ```sh
   terraform apply -var-file=../credentials.tfvars -var-file=global.tfvars
   ```
   {: codeblock}

Once Terraform completes, it will have created:
* a new Cloud Foundry organization
* a `global.env` file under the `outputs` directory in your checkout. This file has environment variables you could reference in other scripts
* the `terraform.tfstate` file

> This tutorial uses the `local` backend provider for Terraform state. Handy when discovering Terraform or working alone on a project, but when working in a team, or on larger infrastructure, Terraform also supports saving the state to a remote location. Given the Terraform state is critical to Terraform operations, it is recommended to use a remote, highly available, resilient storage for the Terraform state  Refer to [Terraform Backend Types](https://www.terraform.io/docs/backends/types/index.html) for a list of available options. Some backends even support versioning and locking of Terraform states.

#### To reuse an organization you are managing

If you are not the account owner but you manage an organization in the account, you can also import an existing organization into Terraform

1. Retrieve the organization GUID
   ```sh
   ibmcloud iam org <org_name> --guid
   ```
   {: codeblock}
1. Change to the `terraform/global` directory
1. Copy [global.tfvars.tmpl](https://github.com/IBM-Cloud/multiple-environments-as-code/blob/master/terraform/global/global.tfvars.tmpl) to `global.tfvars`
   ```sh
   cp global.tfvars.tmpl global.tfvars
   ```
   {: codeblock}
1. Initialize Terraform
   ```sh
   terraform init
   ```
   {: codeblock}
1. After initializing Terraform, import the organization into the Terraform state
   ```sh
   terraform import -var-file=../credentials.tfvars -var-file=global.tfvars ibm_org.organization <guid>
   ```
   {: codeblock}
1. Tune `global.tfvars` to match the existing organization name and structure
1. Apply the changes
   ```sh
   terraform apply -var-file=../credentials.tfvars -var-file=global.tfvars
   ```
   {: codeblock}

### Create per-environment space, cluster and services

This section will focus on the `development` environment. The steps will be the same for the other environments, only the values you pick for the variables will differ.

1. Change to the `terraform/per-environment` folder of the checkout
1. Copy the template `tfvars` file. There is one per environment:
   ```sh
   cp development.tfvars.tmpl development.tfvars
   cp testing.tfvars.tmpl testing.tfvars
   cp production.tfvars.tmpl production.tfvars
   ```
   {: codeblock}
1. Edit `development.tfvars`
   1. Set **environment_name** to the name of the Cloud Foundry space you want to create
   1. Set **space_developers** to the list of developers for this space. **Make sure to add your name to the list so that Terraform can provision services on your behalf.**
   1. Set **cluster_datacenter** to the location where you want to create the cluster. Find the available locations with:
      ```sh
      ibmcloud cs locations
      ```
      {: codeblock}
   1. Set the private (**cluster_private_vlan_id**) and public (**cluster_public_vlan_id**) VLANs for the cluster. Find the available VLANs for the location with:
      ```sh
      ibmcloud cs vlans <location>
      ```
      {: codeblock}
      If you don't have any VLANs, keep the empty values, new VLANs will be created.
   1. Set the **cluster_machine_type**. Find the available machine types and characteristics for the location with:
      ```sh
      ibmcloud cs machine-types <location>
      ```
      {: codeblock}
1. Initialize Terraform
   ```sh
   terraform init
   ```
   {: codeblock}
1. Create a new Terraform workspace for the *development* environment
   ```sh
   terraform workspace new development
   ```
   {: codeblock}
   Later to switch between environments use
   ```sh
   terraform workspace select development
   ```
   {: codeblock}
1. Look at the Terraform plan
   ```sh
   terraform plan -var-file=../credentials.tfvars -var-file=development.tfvars
   ```
   {: codeblock}
   It should report:
   ```
   Plan: 7 to add, 0 to change, 0 to destroy.
   ```
   {: codeblock}
1. Apply the changes
   ```sh
   terraform apply -var-file=../credentials.tfvars -var-file=development.tfvars
   ```
   {: codeblock}

Once Terraform completes, it will have created:
* a Cloud Foundry space
* a Kubernetes cluster
* a database
* a Kubernetes secret with the database credentials
* a storage
* a Kubernetes secret with the storage credentials
* a `development.env` file under the `outputs` directory in your checkout. This file has environment variables you could reference in other scripts
* the environment specific `terraform.tfstate` under `terraform.tfstate.d/development`.

You can repeat the steps for the `testing` and `production`.

### Assign user policies

In the previous steps, roles in Cloud Foundry organization and spaces could be configured with the Terraform provider. For user policies on other resources like the Kubernetes clusters, you are going to rely on the {{site.data.keyword.Bluemix_notm}} CLI `ibmcloud` and the `iam` command.

   ```cmd
   ~/> ibmcloud iam
   NAME:
      ibmcloud iam - Manage identities and access to resources
   USAGE:
      ibmcloud iam command [arguments...] [command options]

   COMMANDS:
      ...
      access-groups                    List access groups under current account
      access-group-create              Create an access group
      access-group                     Show details of an access group
      access-group-delete              Delete an access group
      access-group-update              Update an access group
      access-group-users               List users of an access group
      access-group-user-add            Add user(s) to an access group
      access-group-user-remove         Remove a user from an access group
      access-group-user-purge          Remove user from all access groups
      access-group-service-ids         List service IDs of an access group
      access-group-service-id-add      Add service ID(s) to an access group
      access-group-service-id-remove   Remove a service ID from an access group
      access-group-service-id-purge    Remove service ID from all access groups
      access-group-policies            List policies of an access group
      access-group-policy              Show details of an access group policy
      access-group-policy-create       Create an access group policy
      access-group-policy-update       Update an access group policy
      access-group-policy-delete       Delete an access group policy
      ...
      user-policies                    List policies of a user
      user-policy                      Display details of a user policy
      user-policy-create               Create a user policy for resources in current account
      user-policy-update               Update a user policy for resources in current account
      user-policy-delete               Delete a user policy
      ...
   ```

For the *Development* environment as defined in [this tutorial](./users-teams-applications.html), the policies to define are:

|           | IAM Access policies |
| --------- | ----------- |
| Developer | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Viewer*</li><li>Monitoring: *Administrator, Editor, Viewer*</li></ul> |
| Tester    | <ul><li>No configuration needed. Tester accesses the deployed application, not the development environments</li></ul> |
| Operator  | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Operator*, *Viewer*</li><li>Monitoring: *Administrator, Editor, Viewer*</li></ul> |
| Pipeline Functional User | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Editor*, *Viewer*</li></ul> |

Given a team may be composed of several developers, testers, you can leverage the [access group concept](https://console.bluemix.net/docs/iam/groups.html#groups) to simplify the configuration of user policies. Access groups can be created by the account owner so that the same access can be assigned to all entities within the group with a single policy.

For the *Developer* role in the *Development* environment, this translates to:

   ```sh
   #!/bin/bash

   USER=$1
   GROUP="Example-Developer-Role"

   # Check if the group exist
   if ibmcloud iam access-group $GROUP >/dev/null; then
     echo "Role already exists"
   else
     # Create the access group for the role if the group does not exist
     ibmcloud iam access-group-create $GROUP --description "used by the multiple-environments-as-code tutorial"

     # Set the permissions for this group
     # Resource Group: Viewer
     ibmcloud iam access-group-policy-create $GROUP --roles Viewer --resource-type resource-group --resource "default"

     # Platform Access Roles in the Resource Group: Viewer
     ibmcloud iam access-group-policy-create $GROUP --roles Viewer --resource-group-name "default"

     # Monitoring: Administrator, Editor, Viewer
     ibmcloud iam access-group-policy-create $GROUP --roles Administrator,Editor,Viewer --service-name monitoring
   fi

   # Add the user to the group
   ibmcloud iam access-group-user-add $GROUP $USER
   ```

The [iam/development](https://github.com/IBM-Cloud/multiple-environments-as-code/tree/master/iam/development) directory of the checkout has examples of these commands for the defined *Developer*, *Operator* and *Functional User* roles. To set the policies as defined in a previous section for a user with the *Developer* role in the *development* environment, you can use the script `add-developer.sh`:

   ```sh
   cd iam/development
   ./add-developer.sh user@domain.com
   ```

## Remove resources

1. Activate the `development` workspace
   ```sh
   cd terraform/per-environment
   terraform workspace select development
   ```
   {: codeblock}
1. Destroy the spaces, services, clusters
   ```sh
   terraform destroy -var-file=../credentials.tfvars -var-file=development.tfvars
   ```
   {: codeblock}
1. Repeat the steps for the `testing` and `production` workspaces
1. If you created it, destroy the organization
   ```sh
   cd terraform/global
   terraform destroy -var-file=../credentials.tfvars -var-file=global.tfvars
   ```
   {: codeblock}

## Related content

* [Terraform tutorial](./infrastructure-as-code-terraform.html)
* [Terraform provider](https://ibm-cloud.github.io/tf-ibm-docs/)
* [Examples using {{site.data.keyword.Bluemix_notm}} Provider for Terraform](https://github.com/IBM-Cloud/terraform-provider-ibm/tree/master/examples)
