# Lookup Azure Key Vault Secrets with Ansible

## Step 1 - Create an Azure Key Vault

1. Create a Resource Group with Ansible

    ```yml
    - name: Create resource group
      azure_rm_resourcegroup:
        name: rg-cs-ansible
        location: eastus
    ```

2. Create Azure Key Vault with Ansible

    ```yml
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

    ```bash
    #AzCLI get tenantId
    az account show --subscription "MySubscriptionName" --query tenantId --output tsv

    #AzCLI get objectId
    az ad sp show --id <ApplicationID> --query objectId
    ```

    ```powershell
    #Azure PowerShell get tenantId
    (Get-AzSubscription -SubscriptionName "MySubscriptionName").TenantId

    #Azure PowerSHell get objectId
    (Get-AzADServicePrincipal -ApplicationId <ApplicationID> ).id
    ```

## Step 2 - Create Secrets in Azure Key Vault with Ansible

1. Register Azure Key Vault Info

    ```yml
    - name: Get Key Vault by name
      azure_rm_keyvault_info:
        resource_group: rg-cs-ansible
        name: kv-cs-ansible
      register: keyvault
    ```

2. Set Key Vault URI Fact

    ```yml
    - name: set KeyVault uri fact
      set_fact: keyvaulturi="{{ keyvault | json_query('keyvaults[0].vault_uri')}}"
    ```

3. Create a Secret with Ansible

    ```yml
    - name: Create a secret
      azure_rm_keyvaultsecret:
        secret_name: adminPassword
        secret_value: "P@ssw0rd0987654321"
        keyvault_uri: "{{ keyvaulturi }}"
    ```

## Step 3 - Deploy an Azure Key Vault Resource and Secret with Ansible

```bash
ansible-playbook createKVandSecret.yaml
```

## Step 4 - Lookup Azure Key Vault Secrets with Ansible

### Option 1: Use AzCLI

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
```

```bash
ansible-playbook.yaml lookupSecretAzCLI.yaml
```

### Option 2: Use Azure PowerShell

```yml
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

```bash
ansible-playbook.yaml lookupSecretAzPowerShell.yaml
```

### Option 3: Use azure_keyvault_secret Ansible Lookup Plugin

1. Create lookup_plugin folder

    ```bash
    mkdir lookup_plugins
    ```

2. Download azure_keyvault_secret.py

    ```powershell
    Invoke-WebRequest `
    -Uri 'https://raw.githubusercontent.com/Azure/azure_preview_modules/master/lookup_plugins/azure_keyvault_secret.py' `
    -OutFile lookup_plugins/azure_keyvault_secret.py
    ```

    ```bash
    curl \
    https://raw.githubusercontent.com/Azure/azure_preview_modules/master/lookup_plugins/azure_keyvault_secret.py \
    -o lookup_plugins/azure_keyvault_secret.py
    ```

3. Non-Managed Identity Lookup

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

    ```bash
    ansible-playbook.yaml lookup_azure_keyvault_secret_non-managed-identity.yaml
    ```

4. Managed Identity Lookup

    1. Create an Identity for the Ansible Linux Virtual Machine

        ```powershell
        [Configure managed identities for Azure resources on an Azure VM using PowerShel](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-powershell-windows-vm)
        ```

    2. Add an Access Policy for the Managed Identity

    3. Use azure_keyvault_secret lookup plugin with a Managed Identity

        ```yml
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

        ```bash
        ansible-playbook lookup_azure_keyvault_secret_managed-identity.yaml
        ```
