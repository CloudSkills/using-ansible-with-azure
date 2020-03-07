# Connect to an Azure Windows Host with Ansible

## Step 1 - Add WinRM Support to Ansible

```bash
#from Ansible server
pip install "pywinrm>=0.3.0"
```

## Step 2 - Use Azure VM Extension to Enable HTTPS WinRM Listener

1. Create the playbook

    ```bash
    vi configureWinRMforAzureWindowsVm.yaml
    ```

2. Define the hosts block

    ```yml
    ---
    - hosts: localhost
      connection: local
    ```

3. Create azure_rm_virtualmachineextension task

    ```yaml
        - name: create Azure vm extension to enable HTTPS WinRM listener
          azure_rm_virtualmachineextension:
            name: winrm-extension
            resource_group: rg-cs-ansible
            virtual_machine_name: vm-cs-web01
            publisher: Microsoft.Compute
            virtual_machine_extension_type: CustomScriptExtension
            type_handler_version: '1.9'
            settings: '{"fileUris": ["https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"],"commandToExecute": "powershell -ExecutionPolicy Unrestricted -File ConfigureRemotingForAnsible.ps1"}'
            auto_upgrade_minor_version: true
    ```

4. Custom Script Extension Settings in JSON

    ```json
    {
        "fileUris": ["https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"],
        "commandToExecute": "powershell -ExecutionPolicy Unrestricted -File ConfigureRemotingForAnsible.ps1"
    }
    ```

    [ConfigureRemotingForAnsible.ps1](https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)

## Step 3 - Get the Public Ip Address of the Virtual Machine

```yml
- name: Get facts for one Public IP
    azure_rm_publicipaddress_info:
    resource_group: rg-cs-ansible
    name: pip-cs-web
    register: publicipaddresses
```

## Step 4 - Store the Public Ip Address as an Ansible Fact

1. Parse JSON results

    ```JSON
    {
    "publicipaddresses": [
        {
            "allocation_method": "static",
            "dns_settings": {},
            "etag": "W/\"3784787f-6752-46ff-8640-bf1fa3efcf0e\"",
            "id": "/subscriptions/FAKEsubID-12341234adfasdf2-34341-131dfasd3/resourceGroups/rg-cs-ansible/providers/Microsoft.Network/publicIPAddresses/pip-cs-web",
            "idle_timeout": 4,
            "ip_address": "40.12.123.45",
            "ip_tags": {},
            "location": "eastus",
            "name": "pip-cs-web",
            "provisioning_state": "Succeeded",
            "sku": "Basic",
            "tags": null,
            "type": "Microsoft.Network/publicIPAddresses",
            "version": "ipv4"
        }
    ]
    }
    ```

    ```powershell
    $json  = Get-Content publicipaddressesResults.json | ConvertFrom-Json
    $json.publicipaddresses[0].ip_address
    ```

2. Use set_fact to create a variable

    ```yaml
    - name: set public ip address fact
        set_fact: publicipaddress="{{ publicipaddresses | json_query('publicipaddresses[0].ip_address')}}"
    ```

## Step 6 - Wait for WinRM Connection

```yaml
- name: wait for the WinRM port to come online
    wait_for:
    port: 5986
    host: '{{ publicipaddress }}'
    timeout: 600
```

## Step 7 - Run the Ansible Playbook

```bash
ansible-playbook configureWinRMforAzureWindowsVm.yaml
```
