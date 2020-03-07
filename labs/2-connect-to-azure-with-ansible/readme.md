# Connect to Azure with Ansible

## Prerequisites

* [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7)
* [Azure PowerShell module](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-3.5.0)

## Step 1 - Create an Azure Service Principal

```powershell
$credentials = New-Object Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential `
-Property @{ StartDate=Get-Date; EndDate=Get-Date -Year 2024; Password='<PASSWORD>'};

$spSplat = @{
    DisplayName = 'sp-cs-ansible'
    PasswordCredential = $credentials
}

$sp = New-AzAdServicePrincipal @spSplat
```

## Step 2 - Assign a Role to the Service Principal

```powershell
$subId = ([regex]::Matches((Get-AzContext).Name,'\((.*?)\)')).groups[1].value
#OR
$subId = (Get-AzSubscription -SubscriptionId 'NameOfSubscriptionHere').id

$roleAssignmentSplat = @{
    ObjectId = $sp.id
    RoleDefinitionName = 'Contributor'
    Scope = "/subscriptions/$subId"
}

New-AzRoleAssignment @roleAssignmentSplat
```

## Step 3 - Connect to Azure

_From the Ansible host...._

```bash
docker run -it -w /sln -v "$(pwd)":/sln duffney/ansibleopsbox:latest

#use for credentials file needs access to system
docker run -it -w /sln -v "$(pwd)":/sln --privileged=true duffney/ansibleopsbox:latest
```

### Option 1 - Credentials File

1. Create the credentials file

    ```bash
    mkdir ~/.azure
    vi ~/.azure/credentials
    ```

    _Credential files are used for development environments._

2. Populate the required Ansible variables.

    _Replace `<Text>` with actual values._

    ```bash
    [default]
    subscription_id=<subscription_id>
    client_id=<security-principal-appid>
    secret=<security-principal-password>
    tenant=<security-principal-tenant>
    ```

### Gather Required Azure Information with PowerShell

```powershell
$azConnectInfo = [PSCustomObject]@{
    subscription_id = ([regex]::Matches((Get-AzContext).Name,'\((.*?)\)')).groups[1].value
    client_id = (Get-AzADServicePrincipal -DisplayName 'sp-cs-ansible').ApplicationId
    tenantId = (Get-AzSubscription -SubscriptionId $subId).TenantId
}
$azConnectInfo | fl

```

### Option 2 - Environment Variables

```bash
export AZURE_SUBSCRIPTION_ID=<subscription_id>;
export AZURE_CLIENT_ID=<security-principal-appid>;
export AZURE_SECRET=<security-principal-password>;
export AZURE_TENANT=<security-principal-tenant>;
```

## Create an Azure Resource Group with Ansible

1. Create a playbook file

    ```bash
    vi playbook.yaml
    ```

2. Write the playbook

    ```yml
    ---
    - hosts: localhost
      connection: local
      tasks:
        - name: Create resource group
          azure_rm_resourcegroup:
            name: rg-cs-ansible
            location: eastus
          register: rg
        - debug:
            var: rg
    ```

3. Run the playbook with the ansible-playbook command

    ```bash
    ansible-playbook playbook.yaml
    ```
