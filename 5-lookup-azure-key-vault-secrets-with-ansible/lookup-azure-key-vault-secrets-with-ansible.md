# Lookup Azure Key Vault Secrets with Ansible

## Introduction

As companies become more security conscious, password rotation is often one of the first areas addressed. Secret management is one solution to the problem of password rotation. Decoupling and centralizing secretes into a secret management system is a good start. However, that's not the greatest value add you'll get from a secret management system. The true benefit is being able to rotate those secrets without manual intervention or fear of the unknown side effects of that change.This frees you from having to manually update password and improves security posture.

You accomplish this by first using a secret management system to store secrets and other sensitive information. Second, when writing code that utilizes those secrets it pulls that information at run time from the secrets management system. How this is implemented varies widely and depends on your tooling and coding languages in use. In this tutorial you'll go through the process of storing and retrieving secrets from Azure Key Vault with Ansible.

## Prerequisites

* [Azure subscription](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio)
* [Ansible installed for Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/ansible-install-configure?toc=%2Fazure%2Fansible%2Ftoc.json&bc=%2Fazure%2Fbread%2Ftoc.json#install-ansible-on-an-azure-linux-virtual-machine)
* [Azure PowerShell module](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-3.1.0)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Connect to Azure with Ansible]()

## Step 1 - Create an Azure Key Vault

Before you can use secrets stored in Azure Key Vault you must first create the Azure Key Vault and the secrets you wish to retrieve. Creating these resources in Azure can be done with Ansible. `azure_rm_keyvault` is the Ansible module that is used to create the Azure Key Vault itself. The required parameters are; resource group, sku name, vault tenant, and vault name. While these are the required parameters. The Azure Key Vault will not be useful unless Ansible can access the secrets stored inside. To address that you must also define access policies.

Access to Azure Key Vault can be done in several ways, but for this tutorial you'll use a service principal. This tutorial assumes a service principal as already been created. For more information on creating a service principal reference [Connecting to Azure with Ansible:  Create an Azure Service Principal](https://dev.to/cloudskills/connecting-to-azure-with-ansible-22g2#Create-Azure-Service-Principal). You will need the tenant id and object id of the service principal. Note, that the object id is not the same as the application id or client id referenced in other resources.

```bash
# Get Azure tenantId
az account show --subscription "MySubscriptionName" --query tenantId --output tsv

# Get objectId with AzCLI
az ad sp show --id <ApplicationID> --query objectId
```

```powershell
# Get Azure tenantID with AzPowerShell
(Get-AzSubscription -SubscriptionName "MySubscriptionName").TenantId

# Get objectId with AzPowerShell
(Get-AzADServicePrincipal -ApplicationId <ApplicationID> ).id
```

With the tenantId and service principal objectId you can create the Azure Key Vault. It is worth mentioning that the vault_name value has to be globally unique in Azure. So, don't be alarmed if your name was taken you'll have to find one that hasn't. Another thing to consider is to place the Azure Key Vault in a different resource group than the infrastructure you are building in Azure with Ansible. If you keep in in the same resource group you might accidentally delete it along with other compute resources.

Azure Key Vault can store more than just secrets. It can also store certificates and keys. However, in this tutorial you'll only use it for secrets. Keep in mind the idea of least privilege as you assign permission and ensure the accounts have only the access they need. For this tutorial the service principal will need get, list, and set permissions. Normally, it would only require get and list, but you also have to create the secret within the vault. That tasks can be done with another Ansible module, but requires that the service principal has set permissions on secrets in the Azure Key Vault.

```yaml
---
- hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: rg-cs-ansible
        location: eastus
    - name: Create Ansible Key Vault
      azure_rm_keyvault:
        resource_group: rg-cs-ansible
        vault_name: kv-cs-ansible
        vault_tenant: <tenantID>
        enabled_for_deployment: yes
        sku:
          name: standard
        access_policies:
          - tenant_id: <tenantID>
            object_id: <servicePrincipalObjectId>
            secrets:
              - get
              - list
              - set
```

## Step 2 - Create Secrets in Azure Key Vault

Creating secrets in an Azure Key Vault can be done with the `azure_rm_keyvaultsecret` Ansible module. This Ansible module allows you to create, update, and delete secrets stored in Azure Key Vault. It does not however allow you to lookup those secrets. The azure_rm_keyvaultsecret module requires you specify a secret name, a secret value, and the key vault uri. The key vault uri can be within the portal under the properties blade called DNS Name. It can also be found with AzureCli or AzPowerShell. All of those are valid methods for obtaining the key vault uri, but you can also look it up with Ansible.

```yaml
- name: Create a secret
    azure_rm_keyvaultsecret:
    secret_name: adminPassword
    secret_value: "P@ssw0rd0987654321"
    keyvault_uri: "{{ keyvaulturi }}"
```

The Ansible module `azure_rm_keyvault_info` will query an existing Azure Key Vault and return information about the resource. One of the values returned in the key vault uri. Using the register functionality in Ansible, you can store that information in a variable and parse it with a json query to pull out just the key vault uri. Using `set_fact` you can create another variable containing just the key vault uri. That variable containing the value for the Azure Key Vault uri can then be used to create a secret without having to lookup the uri manually.

```yaml
  - name: Get Key Vault by name
    azure_rm_keyvault_info:
      resource_group: rg-cs-ansible
      name: kv-cs-ansible
    register: keyvault

  - name: set KeyVault uri fact
    set_fact: keyvaulturi="{{ keyvault | json_query('keyvaults[0].vault_uri')}}
```

```yaml
---
- hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Get Key Vault by name
      azure_rm_keyvault_info:
        resource_group: rg-cs-ansible
        name: kv-cs-ansible
      register: keyvault

    - name: set KeyVault uri fact
      set_fact: keyvaulturi="{{ keyvault | json_query('keyvaults[0].vault_uri')}}"

    - name: Create a secret
      azure_rm_keyvaultsecret:
        secret_name: adminPassword
        secret_value: "P@ssw0rd0987654321"
        keyvault_uri: "{{ keyvaulturi }}"
```

_Learn more about [Working with Ansible Register Variables](
https://www.mydailytutorials.com/ansible-register-variables/)._

## Step 3 Option 1 - Use AzPowerShell & AzCLI to Lookup Secrets

At this point you have an Azure Key Vault instance and a secret stored in the vault. That secret can now be used in future playbooks to populate sensitive information. But how do you retrieve that information with Ansible? One way is to use the Azure PowerShell module. Ansible has the ability to run shell commands within a task. On Linux operating systems you'll use the `shell` Ansible module.

By default the shell task will use bash as its command prompt, but using the args parameter you can specify and alternate executable. In order to execute a PowerShell command put in the path to the pwsh executable. In this example you'll specify `/usr/bin/pwsh`, which is the default location for PowerShell on CentOS. In order for you to use the Azure cmdlets in PowerShell you first have to connect to Azure. This requires that you provide a service principal application id and password. As well as a tenant id.

Once connected to Azure you can issue the command to retrieve the Azure Key Vault secret. That cmdlet is `Get-AzKeyVaultSecret`. Get-AzKeyVaultSecret requires that you specify the name of the Azure Key Vault and the name of the secret stored in the vault. The cmdlet does not return the value in text by default. If you enclose the cmdlet with `()` and then add `.SecretValueText` to the end it will output the secret as a string. That string can then be stored in a variable for Ansible to use within the Ansible playbook.

_Read more about using the `shell` Ansible module [here](https://docs.ansible.com/ansible/latest/modules/shell_module.html)._

### Use AzPowerShell

```yaml
---
- hosts: localhost
  connection: local
  vars:
    vaultName: kv-cs-ansible
    vaultSecretName: adminPassword
    servicePrincipalId: <ServicePrincipal.ApplicationId>
    servicePrincipalPassword: P@ssw0rd0987654321
    tenantId: <tenantId>
  tasks:
    - name: connect AzPowerShell to Azure
      shell: |
        $passwd = ConvertTo-SecureString "{{ servicePrincipalPassword }}" -AsPlainText -Force;
        $pscredential = New-Object System.Management.Automation.PSCredential("{{ servicePrincipalId }}", $passwd);
        Connect-AzAccount -ServicePrincipal -Credential $pscredential -Tenant "{{ tenantId }}"
      args:
        executable: /usr/bin/pwsh
    - name: Run a pwsh command
      shell: (Get-AzKeyVaultSecret -Name "{{ vaultSecretName }}" -VaultName "{{ vaultName }}").SecretValueText
      args:
        executable: /usr/bin/pwsh
      register: result
    - debug:
        msg: "{{ result.stdout }}"
```

### Use AzCLI

In case you are not familiar with PowerShell, you can retrieve the same information using Azure CLI. Azure CLI is a command-line interface and will be a familiar experience if you have experience with bash. The general idea is the same. You'll have to connect to Azure using the `az login` command. In order to authenticate with a service principal you'll have to supply the service principal name and the tenant id for that service principal. Once connected to Azure you can retrieve the Azure Key Vault secret using the `az keyvault  secret` command, which requires you provide the Azure Key Vault name and the name of the secret stored in the vault. 

Azure CLI typically outputs JSON data. Which can be queried for specific values. In this example you are only concerned with the property `value` as it contains the secret's value. Also note the `-o tsv` at the end of the command. This ensures the output does not contain `""`. Which is what you want because this value will be used to populate the variable storing the secret value.

_Learn more bout [Querying Azure CLI command output](https://docs.microsoft.com/en-us/cli/azure/query-azure-cli?view=azure-cli-latest)._

```yaml
---
- hosts: localhost
  connection: local
  vars:
    vaultName: kv-cs-ansible
    vaultSecretName: adminPassword
    servicePrincipalId: <ServicePrincipal.ApplicationId>
    servicePrincipalPassword: P@ssw0rd0987654321
    tenantId: <tenantId>
  tasks:
    - name: connect AzPowerShell to Azure
      shell: |
        az login --service-principal -u "{{ servicePrincipalId }}" -p "{{ servicePrincipalPassword }}" --tenant "{{ tenantId }}"
      args:
        executable: /usr/bin/bash
    - name: Run a pwsh command
      shell: az keyvault secret show --name "{{ vaultSecretName }}" --vault-name "{{ vaultName }}" --query value -o tsv
      args:
        executable: /usr/bin/bash
      register: result
    - debug:
        msg: "{{ result.stdout }}"
```

## Step 3 Option 2 - Use azure_keyvault_secret Ansible Lookup Plugin

Lookup Plugins in Ansible are an advanced feature that allow you to  to access data from outside sources.The simplest lookup plugin is one that allows you to access data in a file. Similar to using a `cat` or `Get-Content` command. Microsoft has written an Ansible lookup plugin for Azure Key Vault called `azure_keyvault_secret`. The plugin is written in Python and can be found on GitHub inside the [azure_preview_modules](https://github.com/Azure/azure_preview_modules/blob/master/lookup_plugins/azure_keyvault_secret.py) repository.

{% github Azure/azure_preview_modules/ no-readme %}

The azure_keyvault_secret Ansible lookup plugin is part of an Ansible Galaxy module called [azure_preview_modules](https://galaxy.ansible.com/azure/azure_preview_modules). You can use the lookup plugin by downloading the entire module. However, in this tutorial you'll be adding the lookup plugin directly to your Ansible repo without using the role.

### Add the Lookup Plugin

In order for you to add a lookup plugin to your Ansible code base, you first have to create a directory called `lookup_plugins`. This directory needs to be adjacent to your playbook. If you'd like to specify a different location for it, you can do that by modifying the [ansible.cfg](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings). With the lookup_plugins directory created you next need to create a file with the name of the lookup plugin which is `azure_keyvault_secret.py`. It is a Python script and requires the `.py` extension.

```bash
mkdir lookup_plugins

#PowerShell
Invoke-WebRequest `
-Uri 'https://raw.githubusercontent.com/Azure/azure_preview_modules/master/lookup_plugins/azure_keyvault_secret.py' `
-OutFile lookup_plugins/azure_keyvault_secret.py

#Curl
curl \
https://raw.githubusercontent.com/Azure/azure_preview_modules/master/lookup_plugins/azure_keyvault_secret.py \
-o lookup_plugins/azure_keyvault_secret.py
```

_Read more about [Ansible lookup plugins](
https://docs.ansible.com/ansible/latest/plugins/lookup.html)._

### Non-Managed Identity Lookups

Looking at the examples provided within the comments of the lookup plugin there are two ways to use the lookup plugin; non-managed identity lookups and managed identity lookups. A managed identity is a feature in Azure that provides Azure services with an automatically managed identity in Azure AD. You can use the identity to authenticate to any service that supports Azure AD authentication, including Key Vault, without any credentials in your code. Managed identities are the way to go when your Ansible infrastructure runs in Azure or has the ability to use managed identities. However, that won't always be the case.

When using the azure_keyvault_secret plugin with non-managed identities you have to provide additional information for the plugin to authenticate. In addition to the vault uri and vault secret, you also have to provide a service principal client id, service principal secret, and a tenant id.

```yaml
---
- hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Look up secret when ansible host is general VM
      vars:
        url: 'https://kv-cs-ansible.vault.azure.net/'
        secretname: 'adminPassword'
        client_id: <ServicePrincipal.ApplicationId>
        secret: 'P@ssw0rd0987654321'
        tenant: <tenantId>
      debug: msg="the value of this secret is {{lookup('azure_keyvault_secret',secretname,vault_url=url, client_id=client_id, secret=secret, tenant_id=tenant)}}"
```

_Read more about [managed identities for Azure resources](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)._

### Managed Identity Lookups

In order to use managed identities in Azure for Ansible your Ansible control server will have to be deployed in Azure as a Linux virtual machine. If that is how you have or want your Ansible environment to be, managed identities are the way to go and offer more security and reduce the complicity in code. Deploying an Azure virtual machine and creating a managed identity are out of scope for this tutorial. However, resources are provided below for you to follow if this infrastructure does not already exist in your environment.

__Prerequisites__

1. [Quickstart: Deploy the Ansible solution template for Azure to CentOS](https://docs.microsoft.com/en-us/azure/ansible/ansible-deploy-solution-template)
2. [Configure managed identities for Azure resources on a VM](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm)
3. [Assign managed identity permissions](https://docs.microsoft.com/en-us/azure/key-vault/managed-identity).

Once a managed identity has been created and the permissions assigned to the Azure Key Vault you can query the vault for the stored secrets. Because the authentication is taking place within Azure AD with the managed identity you no longer have to provide the service principal information.

```yaml
---
- hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Look up secret when ansible host is general VM
      vars:
        url: 'https://kv-cs-ansible.vault.azure.net/'
        secretname: 'adminPassword'
      debug: msg="the value of this secret is {{lookup('azure_keyvault_secret',secretname,vault_url=url, client_id=client_id, secret=secret, tenant_id=tenant)}}"
```

## Conclusion

Secret management can either be a nightmare or a blessing. Keep this in mind when implementing infrastructure as code and when authoring automation that requires the use of sensitive information. Keeping those secrets tightly coupled makes the management of the code increasing difficult over time. Seek to decouple the secrets from your code. It will not only make it more manageable, but will also increase your security posture. Imagine how nice it would be to not care or even know if a secret was rotated?
