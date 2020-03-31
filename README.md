# Table of Contents
1. [Understand IaC Cocepts](#IaC)
2. [Understand Terraform's Purpose](#Purpose)
3. [Understand Terraform basics](#Basics)
4. [Use the Terraform CLI (outside of core workflow)](#CLI)
5. [Interact with Terraform modules](#Modules)
6. [Navigate Terraform workflow](#Workflow)
7. [Implement and maintain state](#State)
8. [Read, generate, and modify configuration](#Config)
9. [Understand Terraform Enterprise](#TFE)

Index. https://learn.hashicorp.com/terraform/certification/terraform-associate

# 1. Understand IaC concepts <a name="IaC"></a>
## a. Explain what is IaC
High-level configuration syntax that can be versioned and treated as code. Infrastructure can be shared and re-used. Manage infrastructure using file(s) rather than manually configuring through a UI.
## b. Describe Advantages of IaC Patterns
1. <b>Platform Agnostic</b> - Manage heterogeneous environments with the same workflow by creating a configuration file to fit the needs of a project/org.
2. <b>State Management</b> - Terraform state is the source of the truth by which configuration changes are measured. If a change is made to a configuration, Terraform compares those changes with the state file to determine what changes result in a new resource or modification.
3. <b>Operator Confidence</b> - Is the use of Terraform plan to show what occurs in the environment before it happens.

# 2. Understand Terraform's purpose (vs other IaC) <a name="Purpose"></a>
## a. Explain multi-cloud and provider agnostic benefits
It's often attractive to spread infrastructure across multiple clouds to increase fault-tolerance. By using only a single region or cloud provider, fault tolerance is limited by the availability of that provider. Having a multi-cloud deployment allows for more graceful recovery of the loss of a region or entire provider.
Realizing multi-cloud deployments can be very challenging as many existing tools for infrastructure management are cloud-specific. Terraform is cloud-agnostic and allows a single configuration to be used to manage multiple providers, and to even handle cross-cloud dependencies. This simplifies management and orchestration, helping operators build large-scale multi-cloud infrastructures.

## b. Explain the benefits of state

State is a necessary requirement for Terraform to function. Terraform requires some sort of database to map Terraform config to the real world. When you have a resource `resource "aws_instance" "foo"` in your configuration, Terraform uses this map to know that `instance i-abcd1234` is represented by that resource.

### Metadata
Alongside the mappings between resources and remote objects, Terraform must also track metadata such as resource dependencies.
Terraform typically uses the configuration to determine dependency order. However, when you delete a resource from a Terraform configuration, Terraform must know how to delete that resource. Terraform can see that a mapping exists for a resource not in your configuration and plan to destroy. However, since the configuration no longer exists, the order cannot be determined from the configuration alone.
To ensure correct operation, Terraform retains a copy of the most recent set of dependencies within the state.

### Performance
In addition to basic mapping, Terraform stores a cache of the attribute values for all resources in the state. This is the most optional feature of Terraform state and is done only as a performance improvement.
When running a terraform plan, Terraform must know the current state of resources in order to effectively determine the changes that it needs to make to reach your desired configuration.

### Syncing
In the default configuration, Terraform stores the state in a file in the current working directory where Terraform was run. Okay to get started, but not great when working with a team. 

Remote state is the recommended solution to this problem. With a fully-featured state backend, Terraform can use remote locking as a measure to avoid two or more different users accidentally running Terraform at the same time, and thus ensure that each Terraform run begins with the most recent updated state.

# 3. Understand Terraform Basics <a name="Basics"></a>

Terraform must first be installed on your machine. Terraform is distributed as a binary package for al supported platforms and architectures (macOS, FreeBSD, Linux, OpenBSD, Solaris, Windows). Install Terraform by unzipping it and moving it to a directory included in your system's `PATH`.

## a. Handle Terraform and provider installation and versioning

**What is a Provider?**

A plugin that knows how to talk to a specific set of APIs. Most of the available providers correspond to one cloud or on-prem infrastructure and offer resource types that correspond to each of the features on that platform.
### Provider Configuration
Provider Configuration is created using a `provider` block:
```
provider "google" {
  project = "my-project"
  region  = "us-central1"
}
```

There are two "meta-arguments" that are defined by Terraform itself and available for `provider` blockers:

* `version`, for constraining the allowed provider versions
* `alias`, for using the same provider with different configurations for different resources

### Provider Initialization

Each time a new provider is added to configuration, Terraform must initialize the provider before it can be used. Initialization downloads and installs the provider's plugin so it can be executed.

Provider intialization is one of the actions of `terraform init`. This command will downlad and initialize any providers that are not already installed. Providers downloaded by `terraform init` are only installed for the current working directory.

**NOTE**: `terraform init` cannot automatically download providers that are <b>NOT</b> distributed by HashiCorp. These are the third-party plugins.

### Provider Versions

Provider Versions are released at different times than the core Terraform code and a provider has its own versions. <u>For production, constrain the acceptable version via configuration. Without a constraint, each time terraform init is ran, it will download the latest provider version.</u>
To constrain the provider, add a required_providers block inside of a terraform block:
```
terraform {
  required_providers {
    aws = "~> 1.0"
  }
}
```
When terraform init is re-run with providers already installed, it will use an already-installed provider. Provider version constraints can also be specified using a version arguments within a provider block, but that simultaneously declares a new provider configuration that may cause problems when writing shared modules. Recommendation is to place in the terraform block.

## b. Initialization

The first command to run for a new configuration is `terraform init`. Terraform uses a plugin-based architecture to support hundreds of Infrastructure and service providers. The `terraform init` command downloads and installs providers used within the configuration.

## c. Demonstrate Using Multiple Providers

Multiple provider blocks can exist if a Terraform configuration manages resources from different providers. You can even use multiple providers together. For example you could pass the ID of an AWS instance to a monitoring resource from DataDog.

`alias`: Multiple Provider Instances - define multiple configurations for the same provider, and select which one to use on a per-resource or pre-module. Reason for this is to support multiple regions for a cloud platform, multiple Docker hosts, multiple Consul hosts, etc
```
# The default provider configuration
provider "aws" {
  region = "us-east-1"
}

# Additional provider configuration for west coast region
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```
Selecting alternate providers
```
# Resource block
resource "aws_instance" "foo" {
  provider = aws.west
}

# Module block
module "aws_vpc" {
  source = "./aws_vpc"
  providers = {
    aws = aws.west
  }
}
```
## d. How Terraform finds and fetches providers

Third Party Plugins can be developed by anyone but must be manually installed. `terraform init` does not automatically download them. Place these 3rd party plugins in the user plugins directory. Where depends on the operating system:

| Operating System  | User plugins Directory        |
|-------------------|-------------------------------|
| Windows           | %APPDATA%\terraform.d\plugins |
| All other systems | ~/.terraform.d/plugins        |

Operating system	User plugins directory
Windows	%APPDATA%\terraform.d\plugins
All other systems	~/.terraform.d/plugins

Once a plugin is installed, run `terraform init` to initialize the plugin. Run init from the directory where the configuration files are located.
The same process can be followed for off-line systems. Useful in air-gapped environments and when testing pre-release provider builds.

**Plugin Names and Versions** - The naming scheme for a provider plugin is terraform-provider-_vX.Y.Z, and Terraform uses the name to understand the name and version of a particular provider binary.

Provider Plugin Cache - By default, terraform init downloads plugins into a subdirectory of the working directory so each working directory is self-contained. If you have multiple working directories with the same provider, it will be downloaded again. Alternatively use the shared plugin cache for each distinct binary to be downloaded only once. To enable plugin cache, use plugin_cach_dir in the CLI. The directory must already exist before Terraform will cache plugins, it will not be created for you.
```
# Recommendation
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"

# Alternatively set ENV variable
$ export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```
Once plugin cache is enabled, Terraform will check if the appropriate provider is present in the cache directory and use it. If the specific provider is not in the cache directory, Terraform will download

## e. When and not when to use local-exec or remote-exec

Provisioners are a last resort. Terraform includes the concept of provisioners as a measure of pragmatism, knowing that there will always be certain behaviors that can't be directly represented in Terraform's delcarative model.

Terraform cannot model the actions of provisioners as part of a plan because they can in principle take any action. Secondly, successful use of provisioners requires coordinating many more details than Terraform usage usually requires: direct network access to your servers, issuing Terraform credentials to log in, making sure that all the necessary external software is installed, etc.

Use provisioners only if there is no other options.

**local-exec Provisioner**

The `local-exec` provisioner invokes executable after a resource is created. This invokes a process on the machine running Terraform, not on the resource.

Example usage

```terraform
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "echo ${aws_instance.web.private_ip} >> private_ips.txt"
  }
}
```

**remote-exec Provisioner**

The `remote-exec` provisioner invokes a script on a remote resource after it is created. This can be used to run a configuration management tool, bootstrap into a cluster, etc. `remote-exec` provisioner supports both `ssh` and `winrm`.

Example Usage
```terraform
resource "aws_instance" "web" {
  # ...

  provisioner "remote-exec" {
    inline = [
      "puppet apply",
      "consul join ${aws_instance.web.private_ip}",
    ]
  }
}
```

# 4 Use the Terraform CLI (outside of core workflow)<a name="CLI"></a>


## a. 	Given a scenario: choose when to use `terraform fmt` to format code
Used to to rewrite configuration files to a canonical format and style. `fmt` scans the current directory for config files. If `dir` is used, then it will scan that give directory.
### Usage
```
terraform fmt [options] [DIR]
```

## b. Given a scenario: choose when to use `terraform taint` to taint Terraform resources
If a resource successfully creates but fails during provisioning, Terraform will error and mark the resource as "tainted". A resource that is tainted has been physically created, but can't be considered safe to use since provisioning failed. Terraform will not rollback and destroy the resource when the failure happens because it goes against the execution plan. When a new `terraform plan` is created, Terraform will remove any tainted resources and create new ones.

`taint` to manually mark a terraform managed resource as tainted, forcing it to be destroyed and recreated on the next apply. Command will not modify infrstastructe, but does modify the state file in order to mark a resource as tainted.
### Usage
```
terraform taint [options] address
```

### Example: Taint Single Resource
```
$ terraform taint aws_security_group.allow_all
The resource aws_security_group.allow_all in the module root has been marked as tainted.
```
### Example Tainting a Resource within a Module
```
$ terraform taint "module.couchbase.aws_instance.cb_node[9]"
Resource instance module.couchbase.aws_instance.cb_node[9] has been marked as tainted.
```


## c. Given a scenario: choose when to use `terraform import` to import existing infrastructure into your Terraform state
Terraform can import existing infrastructure. This allows you take resources you've created by some other means and bring it under Terraform management. Terraform import can only import resources into the state, it does not generate configuration. Prior to running `terraform import` it is necessary to write manually a `resource` configuration block for the resource.

### Usage
```
terraform import [options] ADDRESS ID

# Import into Resource
$ terraform import aws_instance.foo i-abcd-1234
```

## d. Given a scenario: choose when to use `terraform workspace` to create workspaces
The "default" workspace is what Terraform starts with and it cannot be deleted. Named workspaces allow conveniently switching between multiple instances of a <i>single</i> configuration within its <i>single</i> backend. Workspaces are equivalent to renaming your statee file. 
### Usage
```
# Create a new workspace to help isolate state
$ terraform workspace new bar

# Select newly created workspace
$ terraform workspace select bar
```

**Note:** The Terraform CLI workspace concept is different from but related to the Terraform Cloud workspace concept.

## e. Given a scenario: choose when to use `terraform state` to view Terraform state

Terraform state is used to map real world resources to your configuration, keep track of metadata, and to improve performance for large infrastructures. This state is stored locally by default in a file named "terraform.tfstate", but can be stored remotely for team engagement.
`terraform state` is used for advanced state management and there are several subcommands `[show][list][mv][pull][push][rm]`. We will focus on the `show` command.

### Example of `terraform show`

```
terraform show [options] ADDRESS

# Show a resource
$ terraform state show 'packet_device.worker'
...
```

### Example of `terraform state list`

Used to list resources within a Terraform state

```terraform
# List all resources including modules
$ terraform state list
aws_instance.foo
aws_instance.bar[0]
module.elb.aws_elb.main
```

### Example of `terraform state pull`

`terraform state pull` is used to manually download and output the state from remote state.

### Example of `terraform state push`

`terraform state push` is used to manually upload a local state file to remote state. Should rarely be used. Meant only as a utility in case manual intervention is necessary with the remote state.

### Example of `terraform state rm`

`terraform state rm` used to remove items from state. Can remove single resources, single instances of a resource, entire modules.

```terraform
$ terraform state rm 'packet_device.worker'
```

## f. Given a scenario: choose when to enable verbose logging and what the outcome/value is
Enable terraform logs by setting the `TF_LOG` environment variable to any value. You can set `TF_LOG` to one of the log levels `TRACE`, `DEBUG`, `INFO`, `WARN` or `ERROR`. `TRACE` is the default and most verbose if `TF_LOG` is set to something other than a log level name.

If Terraform ever crashes (a "panic" in the Go runtime), it saves a log file with the debug logs from the session as well as the panic message and backtrace to crash.log

# 5 Interact with Terraform modules <a name="Modules"></a>

## a. Contrast module source options

<i>Modules</i> in Terraform are self-contained packages of Terraform configurations that are managed as a group. Modules are used to create reusable components, improve organization, and to treat pieces of infrastructure as a black box.

**Finding Modules**

[Terraform Registry](https://registry.terraform.io/) makes it simple to find and use modules. You can search in the Terraform Registry for a module and the resulting modules will be listed. By default, <b>only verified modules are show in the search results.</b> Verified modules are reviewed by HashiCorp to ensure stability and compatibility.

Terraform Registry is integrated into Terraform. Syntax for referencing a registry module is `< NAMESPACE>/< NAME>/< PROVIDER>`. Example: `hashicorp/consul/aws`

`terraform init` will download and cache any modules referenced by a config.

Terraform Cloud provides a private registry for modules. They can be referenced by < HOSTNAME>/< NAMESPACE>/< NAME>/< PROVIDER>.
```
module vpc {
  source = "app.terraform.io/example_corp/vpc/aws"
  version = "0.9.3"
}
```
## b. Interact with module inputs and outputs
A module is a container for multiple resources that are used together. To call a module means to include the contents of that module into a configuration with specific values for its input variables.
```
module "servers" {
  source = "./some-server"
  servers = 5
}
```


Resource outputs can be accessed by printing the attributes at the end using the `output` block. When a resource is nested inside a module you'll need to create an additional level of outputs on a per-module basis.
```
output "server_name" {
  value = module.servers.name
}
```
## c. Describe variable scope within modules/child modules

All modules require a `source` argument. It's value is either the path to a local directory of the module's configuration files or a remote module that TF should download. The input variables serve as parameters for a Terraform module, allowing aspects of the module to be customized without altering the module's own source code. When you declare variables in the root module of your configuration, you can set their values using CLI options and env variables. When you declare in child modules, the calling module should pass values in the module block.
## d. Discover modules from the public Terraform Module Registry

[Terraform Registry](https://registry.terraform.io/)

Every page on the registry has a search filed for finding modules. The search query will look at module name, provider and description. By default, only verified modules are shown in search results. Verified modules are reviewed by Hashicorp. Use filters to view unverified modules.

## e. Defining module version

Recommend to explicitly constraining the acceptable version numbers for each external module to avoid unexpected or unwanted change.

```
module "consul" {
  source = "hashicorp/consul/aws"
  version = "0.0.5"
  ...
}
```
* >= 1.2.0: version 1.2.0 or newer 
* <= 1.2.0: version 1.2.0 or older 
* ~> 1.2.0: any non-beta version >= 1.2.0 and < 1.3.0, e.g. 1.2.X 
* ~> 1.2: any non-beta version >= 1.2.0 and < 2.0.0, e.g. 1.X.Y 
* >= 1.0.0, <= 2.0.0: any version between 1.0.0 and 2.0.0 inclusive 

# 6 Navigate Terraform workflow <a name="Workflow"></a>
## a. Describe Terraform workflow ( Write -> Plan -> Create )

Terraform workflow has three steps:
1. <b>Write</b> - Author infrastructure as code.
1. <b>Plan</b> - Preview changes before applying.
1. <b>Apply</b> - Provision reproducible infrastructure.

### Individual Practitioner

<b>Write</b>

Write Terraform code just like you write code: In an editor of choice. Common to save in a version controlled repository, even working as an individual. As you code away, be sure to run `terraform plan` along the way to flush out any syntax errors.

<b>Plan</b>

When your Terraform development is complete, commit your work to your repo and review the final plan. `terraform plan` will display a plan for confirmation before proceeding to change any infrastructure.

<b>Apply</b>

`terraform apply` will proceed with changing the infrastrucure but only after you confirm the go-ahead. After the infrastructure is declared, commit your changes to your repo.

## b. Initialize a Terraform working directory (terraform init)

`terraform init` command is used to initialize a working directory containing Terraform configuration files. This is the first command that should be run after writing a new Terraform configuration or cloning an existing one from source control.

it's safe to run this command multiple times to bring the working directory up to date with changes in the configuration. This command will never delete existing configuration or state.

### Backend Initialization

During init, the root configuration directory is consulted for backend configuration and the chosen backend is intialized using the given configuration settings. The `-backend-config=...` option can be used for partial backend configuration in situations where the backend settings are dynamic or sensitive and so cannot be statically specified in the configuration file.

### Child Module Installation

During init, the configuration is searched for `module` blocks, and the source code for referenced modules is retrieved fro the locations given in the `source` arguments.

### Plugin Installation

During init, Terraform searches the configuration for both direct and indirect references to providers and attempts to load the required plugins. For providers distributed by HashiCorp, init will automatically download and install plugins if necessary. 

## c. Validate a Terraform configuration (terraform validate)

The `terraform validate` command validates the configuration files in a directory, referring only the configuration and not accessing any remote services such as remote state, provider APIs, etc.

Validate runs checks that verify whether a configuration is syntactically valid and internally consistent, regardless of any provided variables or existing state. It is thus primarily useful for general verification of reusable modules, including correctness of attribute names and value types.

By default, `validate` requires no flags and looks in the current directory for the configurations.

## d. Generate and review an execution plan for Terraform (terraform plan)

`terraform plan` is used to create an execution plan. Terraform performs a refresh, unless explicitly disabled, and then determines what actions are necessary to achieve the desired state specified in the configuration files. The optional `-out` argument can be used to save the generated plan to a file for later executio with `terraform apply`.

### Security Warning

Saved plan files (with the `-out` flag) encode the configuration, state, diff and variables. Variables are often used to store secrets. Therefore, the plan file can potentially store secrets. Terraform itself does not encrypt the plan file. It is highly recommended to encrypt the plan file if you intend to transfer it or keep it at reest for an extended period of time.

## e. Execute changes to infrastructure with Terraform (terraform apply)

`terraform plan` command is used to apply the changes required to reach the desired state of the configuration, or the pre-determened set of actions generated by a `terraform plan` execution plan.

By default, `apply` scans the current directory for the configuration and applies the changes appropriately. However, a path to another configuration or an execution plan can be provided.

## f. Destroy Terraform managed infrastructure (terraform destroy)

`terraform destory` command is used to destroy the Terraform managed infrastructure. Infrastructure managed by Terraform will be destroyed. This will ask for confirmation before destroying.

# 7 Implement and maintain state <a name="State"></a>
## a. Describe default local backend
A "backend" in Terraform determines how state is loaded and how an operation such as apply is executed. This abstraction enables non-local file state storage, remote execution, etc. By default, Terraform uses the "local" backend.

**Benefits of remote backends**
* Working in a team: Backends can store state remotely and protect the state with lcoks to prevent corruption.
* Keep sensitive information off disk.

## b. Outline state locking

Terraform will lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state. Happens automatically on all operations that could write state. If state locking fails, Terraform will not continue. Disable by using command `-lock` but not recommended.

You may use `force-unlock command` to manually unlock the state if unlocking failed.

## c. Handle backend authentication methods

## d. Describe remote state storage mechanisms and supported standard backends

## e. Describe effect of Terraform refresh on state

## f. Describe backend block in configuration and best practices for partial configurations

## g. Understand secret management in state files

Terraform state can contain sensitive data. The state contains resource IDs and all resource attributes. When using local state, state is stored in plain-text JSON files. when using remote state, state is only ever held in memory when used by Terraform. It may be encypted at rest, but this depends on the remote state backend.

Terraform Cloud always encrypts state at rest and protects it with TLS in transit. Terraform Cloud also knows the identity of the user requesting state and maintains a history of state changes. Terraform Enterprise also supports detailed audit logging.

### <b>Recommendations</b>
Storing state remotely can provide better security. Terraform does not persist state to the local disk when remote state is in use, and some backends can be configured to encrypt the state data at rest.
* <u>Terraform Cloud</U> always encrypts state at rest and protects it with TLS in transit. Terraform cloud also knows the identity of the user requesting state and maintains a history of state changes. This can be used to control access and track activity. Terraform Enterprise also supports detailed audit logging.
* The S3 backend supports encryption at rest when enabled. IAM policies and logging can be used to identity any invalid access. Requests for the state go over a TLS connection.

# 8 Read, generate, and modify configuration <a name="Config"></a>
## a.Demonstrate use of variables and outputs

## b.Describe secure secret injection best practice

The Vault provider allows Terraform to read from, write to, and configure HashiCorp Vault.

**IMPORTANT:** Interacting with Vault from Terraform causes any secrets that you read and write to be persisted in both Terraform's state file <i>and</i> in any generated plan files. For any Terraform modules that reads or writes Vault secrets, these files should be treated as sensitive and protected.

### <b>Best Practices</b>

Recommend to avoid placing secrets in your Terraform config or state file wherever possible, and if placed there, ou take steps to reduce and manage your risk. 

### Configure and Populating Vault

Terraform can be used b the Vault administrator to configure Vault and populate it with secrets. In this case, the state and any plans associated with the configuration must be stored and communicated with care, since they will contain in cleartext any values that were writte into Vault.

Terraform has no mechanism to redact or protect secrets that are provided via configuration, so teams choosing to use Terraform for populating Vault secrets should pay careful attention to the notes on each resource's documentation page about how any secrets are persisted to the state and consider carefully whether such usage is compatible with the security policies.

### Using Vault Credentials in Terraform Configuration

Most Terraform providers require credentials to interact iwth a third-party service that they wrap. This provider allows such credentials to be obtained from Vault, which means that operators or systems running Terraform need only access to a suitably-privileged Vault token in order to temp. lease the credentials for other providers. <u>Terraform has no mechanism to redact or protect secrets that are returned via data sources, so secrets read via this provider will be persisted into the Terraform state, into any plan files, and in some cases in the console output produced while planning and applyin.</u>

## c.Understand the use of collection and structural types

## d.Create and differentiate resource and data configuration

## e.Use resource addressing and resource parameters to connect resources together

## f. Use Terraform built-in functions to write configuration

## g. Configure resource using a dynamic block

## h. Describe built-in dependency management (order of execution based)

# 9 Understand Terraform Enterprise capabilities <a name="TFE"></a>

Understand that there are three different offerings.
* OSS CLI, which is a free and open source production. This is what individuals and small teams use first to get accustom to Terraform.
* Terraform Cloud allows for some extra features noted below, but also allows for better team collaboration and extends the use of Terraform to other teams and departments.
* Terraform Enterprise offers everything in Terraform Cloud, but a way to install in the local datacenter for airgap purpose and additional enterprise features like auditing and SSO integration.

| CLI | Terraform Cloud | Terraform Enterprise |
|------|-------------------|----------------------|
| VCS Integration | VCS Integration | VCS Integration |
| Workspace Management | Workspace Management | Workspace Management |
| Secure Variable Storage | Secure Variable Storage | Secure Variable Storage |
| Remote Runs & Applies |  Remote Runs & Applies |  Remote Runs & Applies |
| Full API Coverage |  Full API Coverage |  Full API Coverage |
| Private Module Registry | Private Module Registry | Private Module Registry |
|   | Roles / Team Management | Roles / Team Management |
|   | Sentinel (paid) | Sentinel |
|   | Cost Estimation | Cost Estimation |
|   |    | <b>SAML / SSO</b> |
|   |    | <b>Clustering</b> |
|   |    | Private DC Installation |
|   |    | Private Network Connectivity |
|   |    | Self-Hosted |
|   |    | <b>Audit Logs</b> |

## a. Terraform Cloud Overview

Terraform Cloud is a platform that performs Terraform runs to provision infrastructure, either on demand or in response to various events. Unlike a general-purpose continuous integration (CI) system, it is deeply integrated with Terraform's workflows and data, which allows it to make Terraform significantly more convenient and powerful.

### VCS Integration

VCS, version control system, provides additional features and improved workflows like:
* TF cloud can automatically initiate Terraform runs when a change is committed.
* TF Cloud makes code review easier by automaticaly predicting how pull requests will affect infrastructure.
* Publish new versions of a private Terraform module by pushing a tag to the mod's repo.

| Supported VCS Providers |  |
| ------------------------|---|
| Github.com | Github.com (OAuth) |
| Github Enterprise | GitLab.com |
| GitLab EE and CE | Bitbucket Cloud |
| Bitbucket Server | Azure DevOps Serve | 
| Azure DevOps Services|  |

### Workspace Contents
Terraform Cloud workspaces and local working directories serve the same purpose, but they store their data differently:

| Component               | Local Terraform                                             | Terraform Cloud                                                            |
|-------------------------|-------------------------------------------------------------|----------------------------------------------------------------------------|
| Terraform configuration | On disk                                                     | In linked version control repository, or periodically uploaded via API/CLI |
| Variable values         | As .tfvars files, as CLI arguments, or in shell environment | In workspace                                                               |
| State                   | On disk or in remote backend                                | In workspace                                                               |
| Credentials and secrets | In shell environment or entered at prompts                  | In workspace, stored as sensitive variables |

### Private Module Registry

Terraform Cloud's private module registry helps you share Terraform modules across your organization. It includes support for module versioning, a searchable and filterable list of available modules, and a configuration designer to help you build new workspaces faster. Works a lot like the public Terraform registry.

### Sentinel Overview

Sentinel is an embedded policy-as-code framework integrated with the HashiCorp Enterprise products. It enables fine-grained, logic-based policy decisions, and can be extended to use information from external sources.

### Cost Estimation

Terraform Cloud provides cost estimates for many resources found in your Terraform configuration. For each resource an hourly and monthly cost is shown, along with the monthly delta. The total cost and delta of all estimable resources is also shown.

Cost estimations are supported for <b>AWS, GCP and Azure</b>.

## b. Terraform Enterprise Overview

Terraform Enterprise is a self-hosted distribution of Terraform Cloud. It offers enterprises a private instance of the Terraform Cloud application, with no resource limits and with additional enterprise-grade architectural features like audit logging and SAML single sign-on.

### <u>Deployment Method</u>

There are two ways to install TFE

1. <b>Cluster Deployment</b> Deploy TFE as a cluster of three or more instances (+100) using a Terraform module. The cluster's secondary insances can scale horizontally to fit your workloads.
1. <b>Individual Deployment</b> Deploy TE directly on a Linux instance using an executable installer.

### Data Storage

Make sure your data storage services or device meet Terraform Enterprise's requirements. These requirements differ based on operational mode:

* <b>External Services:</b>

  * PostgreSQL
  * Any S3-compatible object storage service (Azure Blob). Create bucket before install and specifcy bucket during installation. Be sure to put bucket in same region as instance.
  * Optionally: If already running Vault, configure TFE to use that instead of running its own Vault instance.

* <b>Mounted disk:</b>
  
  * Mounted disk requirements

    * If you choose "Production - Mounted disk" operational mode, Terraform Enterprise will manage its own PostgreSQL database and object storage using a separate directory on the host

### Linux Instance

| Cluster Deployment | Individual Deployment |
| -------------------| ----------------------|
| Ubuntu 16.04 / 18.04 | Debian 7.7+ |
| Red Hat Enterprise 7.4-7.7 | Ubuntu 14.04/16.04/18.04 |
| CentOS 7.4 - 7.7 | RHEL 7.4 - 7.7 |
|  | CentOS 6.x / 7.4 - 7.7 |
|  | Amazon Linux Distro |
|  | Oracle Linux 7.4 - 7.7 |

### Hardware Requirements

| Hardware |
| ---------|
| At least 40GB of disk space on root volume |
| At least 8GB of RAM |
| At least 2 CPU cores |

### <u>Architecture of Cluster Deployment</u>

A Terraform Enterprise cluster consists of two types of servers: primaries and secondaries (also called workers). The primary instances run additional, stateful services that the secondaries do not.

There should always be three primary nodes to ensure cluster stability, but the cluster can scale by adding or removing secondary nodes.

Clustered deployment relies on a Terraform module to provision infrastructure. Cluster size is controlled by the module's input variables, and the number of secondary instances can be changed at any time by editing the variables and re-applying the configuration

### Load Balancing

A Terraform Enterprise cluster relies on a load balancer to direct traffic to application instances.

### Data Storage

Clustered deployment is designed for use with external data services. These include a PostgreSQL database and a blob storage service.



## a. Describe the benefits of Sentinel, Registry, and Workspaces
* <b>Sentinel</b> - Policy as code framework by Hashicorp. A policy-oriented language to write policies, and integrates with TFE and Nomad Enterprise for enforcement.
* <b>Module Registry</b> - Terraform cloud's private module registry to share TF modules across organization. Support for module versioning, search and fitlerable list of available modules and a configuration designer.
* <b>Workspaces</b> - Terraform Cloud manages infrastructure collections with <i>workspaces</i> instead of directories. A workspace contains everyting Terraform needs to managed a given collection of infrastructure.

<b>NOTE:</b> Terraform CLI workspaces alternate state files in the same working directory; they're a convenience feature for using one configuration to manage multiple similar groups of resources.



Terraform Cloud keep some additional data for each workspace:
* <b>State Versions</b> - Each workspace retains backups of its previous state files
* <b>Run History</b> - TF Cloud retains a record of all run activity.

## b. Differentiate OSS and TFE workspaces
* CLI Workspaces
* Enterprise / Cloud Workspaces
## c. Summarize features of Terraform Cloud

## Overview of Terraform Cloud

Terraform Cloud is offered as a multi-tenant SaaS platform and is deisgned to suit the needs of smaller teams and organizations. It is limited to one run at a time, which prevents users from executing multiple run concurrently.

**The Application**
Terraform cloud located at https://app.terraform.io, provides a UI and API to manage Terraform projects. Manageds projects in terms of organizations and workspaces:

* <b>Workspace</b> is a named container for a single timeline of Terraform state, used to manage a collection of infrastructure resources over time. Each workspace belongs to an org, and only members of that org can acces it.
* <b>Organization</b> is a group of users who can collaborate on a shared set of workspaces. An organizations is created by an initial user, who can then add others.

**Features**

Terraform cloud offers free and paid for features.

 * Integrate with most popular version control systems.
 * Manage your project's state, including state locking.
 * Plan and apply configuration changes from within the Terraform Cloud UI.
 * Securely store variables, including secret values.
 * Store and use private Terraform modules.
 * Collaborate with other users.
