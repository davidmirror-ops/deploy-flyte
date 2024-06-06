# Introduction

The modules in this repo have been tested with Terraform and OpenTofu, and will help you deploy a production-grade Flyte instance on Microsoft Azure. 

# Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform) (version 1.3.7)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli#install)
- [Helm](https://helm.sh/docs/intro/install/#through-package-managers)
- Your Microsoft account should have access to an Azure subscription with at least Contributor role.
- Log into Azure via `az login`


# Configure the Terraform backend

- Once logged in, create a new:
    - Resource Group
    - Storage account with default settings
    - Storage container for the Terraform state
- Put these values into [backend.tfvars](./backend.tfvars)

# Update locals values

- Go to [locals.tf](./locals.tf) and update the values to match your desired configuration.


# Create Cluster & Cluster Resources
1. From the `environments/azure/flyte-core` folder, initialize the Terraform backend:

```bash
cd environments/azure/flyte-core && terraform init -backend=true -backend-config=backend.tfvars
```
2. Generate a Terraform plan:

```bash
terraform plan -out=flyte.plan
```
3. Apply the plan:
```bash
terraform apply flyte.plan
```
Example output:
```bash
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "flytedeploy01.eastus.cloudapp.azure.com"
```

# Testing your deployment

1. Verify Flyte's backend status


```bash
kubectl get pods -n flyte

NAME                                 READY   STATUS    RESTARTS   AGE
datacatalog-6864645db6-99msb         1/1     Running   0          6m45s
flyte-pod-webhook-848d7db899-8wltj   1/1     Running   0          6m45s
flyteadmin-6cc67b49b4-cmt7j          1/1     Running   0          6m45s
flyteconsole-68f677797f-p4s98        1/1     Running   0          6m45s
flytepropeller-b88f7bf6d-lqc8s       1/1     Running   0          6m45s
flytescheduler-844db4658c-hfrhv      1/1     Running   0          6m45s
syncresources-767d7fc77b-5mj6n       1/1     Running   0          6m45s
```
2. Update your `$HOME/.flyte/config,yaml` and configure `endpoint` with the value of the `cluster_endpoint` output:
>NOTE: installing `flytectl` will typically create an initial `config.yaml` file. [Learn more](https://docs.flyte.org/projects/flytectl/en/latest/#installation).

Example:
```yaml
...
admin:
  endpoint: dns:///flytedeploy01.eastus.cloudapp.azure.com" 
  insecure: false #it means, the connection uses SSL, even if it's a temporary cert-manager cert.
...
#Uncomment only if you want to test CLI commands and the certificate is not generated yet.
# You can confirm the cert by either going to the UI (a valid certificate should be used) or
#from your terminal: kubectl get challenges.acme.cert-manager.io -n flyte (there should not be any pending challenge). With this flag enabled, SSL is still used but the client doesn't verify the certificate chain.

  #insecureSkipVerify: true 
```
> NOTE: this configuration step is only needed for CLI access (`flytectl` or `pyflyte`), not for the UI.


3. Save the following "hello world" workflow definition:
```bash
cat << 'EOF' >hello_world.py
from flytekit import task, workflow
@task
def say_hello() -> str:
    return "hello world"
@workflow
def my_wf() -> str:
    res = say_hello()
    return res
if __name__ == "__main__":
    print(f"Running my_wf() {my_wf()}")
EOF
```
4. Execute the workflow on the Flyte cluster:
```bash
pyflyte run --remote hello_world.py my_wf
```
Example output:
```bash
Running Execution on Remote.

[✔] Go to https://flytedeploy01.eastus.cloudapp.azure.com/console/projects/flytesnacks/domains/development/executions/fae18cf6750bd4d64bc7 to see execution in the console.
```
5. Go to the console and verify the succesful execution:

![](https://raw.githubusercontent.com/flyteorg/static-resources/main/common/flyte-on-azure-execution.png)  

**Congratulations!**  
You have a fully working Flyte environment on Azure.

From this point on, you can continue your learning journey by going through the [Flyte Fundamentals](https://docs.flyte.org/en/latest/flyte_fundamentals/index.html) tutorials.

## How to tear down your deployment

1. Once you're finished testing/using Flyte, just invoke the following command:

```bash
terraform destroy
```
