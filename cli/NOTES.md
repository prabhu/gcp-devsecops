# Introduction

[gcloud](https://cloud.google.com/sdk/gcloud/) is an important part of the DevSecOps toolchain. This page uses scenarios and examples to explain the commands and the options.

## #1 Use configuration

Do not use the default configuration for more than a week. To create a new [configuration](https://cloud.google.com/sdk/docs/configurations) use the below command

```bash
gcloud config configurations create project-devops

gcloud config list
```

To set various properties use the `config set` sub-command.

```bash
# Sets the default compute zone
gcloud config set compute/zone europe-west2

# Sets the default EKS cluster name
gcloud config set container/cluster dev-eks-cluster

gcloud config set disable_usage_reporting True
gcloud config set survey/disable_prompts True
```

To automate activating the right configuration for the given project, use a tool such as [direnv](https://direnv.net/)

```bash
echo CLOUDSDK_ACTIVE_CONFIG_NAME=project-devops >> .envrc
```

### Some properties to consider

- disable_file_logging: Disable cli logs for CI pipeline based invocations
- user_output_enabled: Disable output for commands
- billing/quota_project: If you need to operate on one project, but need quota against a different project
- builds/use_kaniko: Enables [kaniko](https://cloud.google.com/cloud-build/docs/kaniko-cache) cache
- compute/use_new_list_usable_subnets_api: If True, use the new API for listing usable subnets which only returns subnets in the current project
- auth/impersonate_service_account: Service account email to impersonate for all cli invocations. Useful to enforce a permission boundary for cli access.
 
To view all available configuration properties use the command below which is more upto date than the official documentation

```bash
gcloud topic configurations
```

## #2 Use Access Context Manager

Don't trust the old and dated perimeter security model. [Access Context Manager](https://cloud.google.com/access-context-manager/docs/overview) is security done properly for cloud!

### Get Policy Editor permission

```bash
gcloud organizations add-iam-policy-binding ORGANIZATION_ID \
  --member="user:example@customer.org" \
  --role="roles/accesscontextmanager.policyEditor"
``` 

### Create an access policy for the organization

```bash
gcloud access-context-manager policies create \
    --organization ORGANIZATION_ID --title org-default

# List the policies
gcloud access-context-manager policies list \
    --organization ORGANIZATION_ID
```

### Create access levels

Use [YAML specification](https://cloud.google.com/access-context-manager/docs/example-yaml-file) with basic-level-spec parameter

```bash
gcloud access-context-manager levels create Device_Trust \
    --basic-level-spec=corpdevspec.yaml \
    --combine-function=AND \
    --description='Access level that conforms to corporate spec.' \
    --title='Device_Trust Policy limits to Mac and Windows in GB'

# List the access level
gcloud access-context-manager levels list --policy=Device_Trust
```

```yaml
- devicePolicy:
    allowedEncryptionStatuses:
      - ENCRYPTED
    osConstraints:
      - osType: DESKTOP_MAC
      - osType: DESKTOP_WINDOWS
    requireScreenlock: true
    ipSubnetworks:
      - 252.0.2.0/24
    regions:
      - GB
    members:
      - user:exampleuser@example.com
      - serviceAccount:exampleaccount@example.iam.gserviceaccount.com

- ipSubnetworks:
    - 176.0.2.0/24
```
