# Introduction

Ansible is an agentless configuration management tool. Instead of using an agent to communicate with target machine Ansible relies on remote management protocols such as SSH and WinRM. Not depending on an agent for connectivity has it's pros can cons. While you do not have to install an agent, you do have to setup the remote management protocol. In this tutorial you'll be provisioning a Windows Server virtual machine by using PowerShell and a custom script extension Deployed by Ansible to Azure. By the end of the tutorial, you will be able to connect to the Azure virtual machine with Ansible using WinRM.

# Table Of Contents

* [Prerequisites](#prerequisites)
* [Step 1 - Add WinRM Support to Ansible](#step-1-add-winrm-support-to-ansible)
* [Step 2 - Configure the WinRM Listener](#step-2-configure-the-winrm-listener)
* [Step 3 - Use Azure VM Extension to Enable HTTPS WinRM Listener](#step-3-use-azure-vm-extension-to-enable-https-winrm-listener)
* [Step 4 - Get the Azure Public Ip Address of the Virtual Machine](#step-4-get-the-azure-public-ip-address-of-the-virtual-machine)
* [Step 5 - Store the Public Ip Address in an Ansible Fact](#step-5-store-the-public-ip-address-in-an-ansible-fact)
* [Step 6 - Wait for WinRM Connection](#step-6-wait-for-winrm-connection)
* [Step 8 - Configure Azure Virtual Machine to Connect to Ansible via WinRM](#step-7-configure-azure-virtual-machine-to-connect-to-ansible-via-winrm)
* [Conclusion](#conclusion)

## Prerequisites <a name="prerequisites"></a>

In order to follow along with this tutorial you'll need your Ansible environment connected Azure. All the required resources for an Azure virtual machine must also be deployed prior to following this tutorial. Both of these prerequisites are covered previously in this series.

* [Connecting to Azure with Ansible]()

* [Deploying Resources to Azure with Ansible]()

## Step 1 - Add WinRM Support to Ansible <a name="step-1-add-winrm-support-to-ansible"></a>

Windows remote management (WinRM) is a management protocol used by Windows to remotely communicate with another server. Ansible uses this protocol to communicate to Windows targets. In order to use WinRM you must configure the Ansible server to support WinRM traffic and configure the Windows host. The Windows host configuration depends on your environment and how you want to configure the WinRM listener.

In order for Ansible to communicate over WinRM requires the [pywinrm](https://github.com/diyan/pywinrm) packaged be installed on the Ansible server. You can install it by running the command `pip install "pywinrm>=0.3.0"`. The pywinrm package is all that is required to be installed on the Ansible server.

```shell
pip install "pywinrm>=0.3.0"
```

_Read more about [Windows Remote Management](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#windows-remote-management)_

## Step 2 - Configure the WinRM Listener <a name="step-2-configure-the-winrm-listener"></a>

The configuration of a WinRM listener has two main pieces to configure. The port it uses to communicate with and the authentication option used. In this tutorial you'll be configuring the WinRM listener to use port 5986 and you will authenticate with NTLM. Using port 5986 requires the use of certificates for encryption. There are several ways to deploy the certificates to the windows host, but you'll be using self signed certificates in this tutorial. Ansible provides a PowerShell script that will configure all of this for you. It will be with an Azure custom vm extension to automate the configuration through Ansible.

_Read more about [Setting up a Windows Host](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#setting-up-a-windows-host)._

[ConfigureRemotingForAnsible.ps1](https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)

_The ConfigureRemotingForAnsible.ps1 script is intended for training and development purposes only and should not be used in a production environment, since it enables settings (like Basic authentication) that can be inherently insecure._

_Location in the repo is `/examples/scripts/ConfigureRemotingForAnsible.ps1`._

{% github https://github.com/ansible/ansible %}

## Step 3 - Use Azure VM Extension to Enable HTTPS WinRM Listener <a name="step-3-use-azure-vm-extension-to-enable-https-winrm-listener"></a>

Before Ansible can communicate with the Azure Windows virtual machine hosted in Azure the WinRM listener must be configured. In this tutorial you are configuring the WinRM listener to use port 5986 which uses self signed certificates for encryption. You will also use NTLM for your authentication as this virtual machine is not yet part of an Active Directory domain. All of the configuration is handled by a PowerShell script called `ConfigureRemotingForAnsible.ps1`. The next step is to have Ansible run that script on the Azure virtual machine. Delegating that task to the Azure virtual machine isn't an option because WinRM is not yet configured. However, Azure offers a [custom script extensions](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-windows) resource that will allow you to delegate the task of running the `ConfigureRemotingForAnsible.ps1` to Azure.

### Using Azure VM Extension to Enable HTTPS WinRM Listener <a name="using-azure-vm-extension-enable-https-winrm-listener"></a>

Before Ansible can communicate with the Azure Windows virtual machine hosted in Azure the WinRM listener must be configured. In this tutorial you are configuring the WinRM listener to use port 5986 which uses self signed certificates for encryption. You will also use NTLM for your authentication as this virtual machine is not yet part of an Active Directory domain. All of the configuration is handled by a PowerShell script called `ConfigureRemotingForAnsible.ps1`. The next step is to have Ansible run that script on the Azure virtual machine. Delegating that task to the Azure virtual machine isn't an option because WinRM is not yet configured. However, Azure offers a [custom script extensions](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-windows) resource that will allow you to delegate the task of running the `ConfigureRemotingForAnsible.ps1` to Azure.

Ansible's `azure_rm_virtualmachineextension` module allows you to create virtual machine custom script extensions. This module requires that you specify a name, resource_group, virtual_machine_name, and publisher. These parameters are all used to target the correct Azure virtual machine. The parameters; virtual_machine_extension_type, type_handler_version, and auto_upgrade_minor_version define and configure the extension being created. Which in this tutorial is a CustomScriptExtension. The settings parameter is where you instruct the CustomScriptExtension on what to do.

The problem you are attempting to solve with the custom script extension is to run the `ConfigureRemotingForAnsible.ps1` on the newly deploy Azure virtual machine. Before the PowerShell script can be run, you'll have to download it. The Azure customs script extension allows you to do that by using the `fileUris` property. Once downloaded the script needs to be executed. You will use the `commandToExecute` property to specify the executable and the parameters for the executable. In this example the executable is PowerShell. The parameters used are `ExecutionPolicy` and `File`. [ExecutionPolicy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-6)  is set to Unrestricted which will allow you to run the script without modifying PowerShell's execution policies on the machine. File specifies the file name of the script to be executed. It assumes a relative path that fileUris used to download the script. Which means you do not need to specify the full path. These settings are stored in JSON and will be used by the settings parameter of the Ansible task.

_Read more about [Azure Custom Script Extensions](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-windows)._

```json
{
    "fileUris": ["https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"],
    "commandToExecute": "powershell -ExecutionPolicy Unrestricted -File ConfigureRemotingForAnsible.ps1"
}
```

To create an custom script extension resource in Azure you'll use the [azure_rm_virtualmachineextension](https://docs.ansible.com/ansible/latest/modules/azure_rm_virtualmachine_module.html) Ansible module. In order to create it, you must specify a name for the custom script extensions `winrm-extension`, a resource group `ansible_rg`, the virtual machine the extension will be attached to `vm-cs-web01`, the publisher `Microsoft.Compute`, virtual machine extensions type `CustomScriptExtension`, type handler version `1.9`, settings for the custom script extension using JSON, and define if the extension should auto upgrade minor versions.

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

_This task will create a custom script extension for the Azure virtual machine vm-cs-web01 that downloads the `ConfigureRemotingForAnsible.ps1` script and then executes it using PowerShell._

## Step 4 - Get the Azure Public Ip Address of the Virtual Machine <a name="step-4-get-the-azure-public-ip-address-of-the-virtual-machine"></a>

Creating the custom script extension that runs ConfigureRemotingForAnsible.ps1 is all that you need to setup the WinRM listener. However, you will not be able to continue with configuring the virtual machine until the listener is responding. Ansible has the ability to wait for a connection, but before you can do that you'll need the public Ip address of the virtual machine. You can gather this information from Azure with Ansible using the `azure_rm_publicipaddress_info` module. All you need to specify is the resource group name and the name of the public ip address resource. To store the output of the module you'll use an Ansible feature called register. `Register` will store the output from the azure_rm_publicipaddress into a variable called publicipaddresses.

```yaml
    - name: Get facts for one Public IP
      azure_rm_publicipaddress_info:
        resource_group: rg-cs-ansible
        name: pip-cs-web
      register: publicipaddresses
```

## Step 5 - Store the Public Ip Address in an Ansible Fact <a name"step-5-store-the-public-ip-address-in-an-ansible-fact"></a>

The variable publicipaddresses contains a lot more than just the public Ip address. It contains other information about the public ip address you don't need. Such as the allocation method, location, and sku.

_publicipaddresses variable_

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

Since the publicipaddresses variable is a JSON object you can query it to get just the information you need. You can accomplish this by using the `set_fact` Ansible module, which allows you to populate variables at run time. Ansible also provides several methods you can use to [filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html) variables and other data. Since the variable you're working with is JSON, you'll use the [JSON Query Filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#json-query-filter). Using set_fact you'll set a create a new variable called `publicipaddress`. The value will be set by querying the publicipaddress variable. The JSON query takes the first item in the publicipaddress array and selects the ip_address property from that arrary. The `ip_address` property containers the public ipv4 Ip address of the virtual machine. With the public Ip address store in a variable the last task to create is a wait for connection task which waits for the custom script extension to setup WinRM.

```yaml
    - name: set public ip address fact
      set_fact: publicipaddress="{{ publicipaddresses | json_query('publicipaddresses[0].ip_address')}}"
```

## Step 6 - Wait for WinRM Connection <a name"step-6-wait-for-winrm-connection"></a>

When authoring Infrastructure as Code documents, you'll often encounter scenarios that require you to wait for a connection to become available. Some examples of that are after a virtual machine reboots or in this tutorial waiting for a script to execute that configures remote management of the virtual machine. Ansible provides a module that allows you to do this called `wait_for`. wait_for has a lot of flexibility but has requires three basic pieces of information. The `port` to communicate on the `host` to attempt to connect to and the `timeout` in seconds. 

In this tutorial you'll wanting to connect to an Azure virtual machine on port `5986`. The host information has been populated into a variable called `publicipaddress` and the timeout you'll set it to `600` seconds or 10 minutes.  

```yaml
    - name: wait for the WinRM port to come online
      wait_for:
        port: 5986
        host: '{{ publicipaddress }}'
        timeout: 600
```

## Step 7 - Configure Azure Virtual Machine to Connect to Ansible via WinRM <a name="step-7-configure-azure-virtual-machine-to-connect-to-ansible-via-winrm"></a>

In this final step the only thing left is to chain the Ansible tasks together in a playbook and execute the playbook. The Ansible playbook contains two sections `hosts` and `tasks`. hosts specifies where and how to run the playbook. `localhost` defines the machine to run the playbook on. Which is the Ansible server. Setting the `connection` to `local` executes the playbook locally on the Ansible server vs. running the playbook over SSH or WinRM.

The `tasks` section defines all the plays Ansible will execute and the order in which they are executed. This playbook starts off by deploying an Azure Custom Script Extension to a virtual machine to configure WinRM. After that it gathers information about the virtual machine's public Ip address and sets an Ansible variable called publicipaddress containing the public ip address of the Azure virtual machine. The last tasks waits for a connection to the virtual machine on port 5986. If it does not succeed after 600 seconds it will timeout and fail. This step is used to verify the Azure Custom Script Extension was successful.

```yaml
#configureAzureWinVmWinRM.yaml
---
- hosts: localhost
  connection: local

  tasks:
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

    - name: Get facts for one Public IP
      azure_rm_publicipaddress_info:
        resource_group: rg-cs-ansible
        name: pip-cs-web
      register: publicipaddresses

    - name: set public ip address fact
      set_fact: publicipaddress="{{ publicipaddresses | json_query('publicipaddresses[0].ip_address')}}"

    - name: wait for the WinRM port to come online
      wait_for:
        port: 5986
        host: '{{ publicipaddress }}'
        timeout: 600
```

![provision azure resources ansible gif](https://thepracticaldev.s3.amazonaws.com/i/ctedozxtcixa2ukctxyz.gif)

```bash
ansible-playbook configureAzureWinVmWinRM.yaml
```

### Conclusion

All configuration management tools require some sort of provisioning to be done in order to connect to remote machines. Even agentless configuration management tool such as Ansible require some provisioning in order to communicate with the machines using a remote management protocol. However, it is possible to define the provisioning steps and task in Ansible by leveraging automation provided by other platforms. Doing so allows you to keep the entire configuration in one place, such as an Ansible playbook. Keeping it all in one place or within one tool keeps it simple and clean.
