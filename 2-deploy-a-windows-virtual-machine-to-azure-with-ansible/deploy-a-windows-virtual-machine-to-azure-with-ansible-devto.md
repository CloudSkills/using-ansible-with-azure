# Introduction

Ansible is a configuration management tool used to control and apply configuration changes to infrastructure. However, before you can apply changes to the infrastructure it first has to exist. Ansible has several Azure modules that allow you to deploy resources in Azure. This tutorial will walk you through deploying a Windows virtual machine to Azure using an Ansible playbook. By the end of the tutorial you'll understand how to use several of the Azure Ansible modules to deploy workloads to Azure.

## Table Of Contents

* [Prerequisites](#prerequisites)
* [Step 1 - Create a Resource Group](#step-1-create-a-resource-group)
* [Step 2 - Create a Virtual Network](#step-2-create-a-virtual-network)
* [Step 3 - Create a Public Ip](#step-3-create-a-public-ip)
* [Step 4 - Create a Network Security Group](#step-4-create-a-network-security-group)
* [Step 5 - Create a Virtual Network Interface Card](#step-5-create-a-virtual-network-interface-card)
* [Step 6 - Create a Virtual Machine](#step-6-create-a-virtual-machine)
* [Step 7 - Deploy an Azure Windows Virtual Machine](#step-7-deploy-an-azure-windows-virtual-machine)
* [Conclusion](#conclusion)

## Prerequisites <a name="prerequisites"</a>

In order to follow along in this tutorial you'll need the following:

* [Ansible installed](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/ansible-install-configure?toc=%2Fazure%2Fansible%2Ftoc.json&bc=%2Fazure%2Fbread%2Ftoc.json#install-ansible-on-an-azure-linux-virtual-machine)
  * `ansible[azure]` pip packaged installed
* [Connect to Azure with Ansible]()

### Ansible Modules Required to Deploy an Azure Virtual Machine

Creating a virtual machine in Azure requires several different  Azure resources; a resource group, virtual network, subnet, public ip address, network security group, network interface card, and the virtual machine itself. Each of these Azure resources can be managed and modified using an Ansible module. These Ansible modules allow you to codify your infrastructure in yaml files in the form of Ansible playbooks. Below is a list of the Ansible modules that are used throughout this tutorial. However, many more Ansible modules exist and can be found in [Ansible Module Index.](https://docs.ansible.com/ansible/latest/modules/list_of_cloud_modules.html#azure) You do not have to install all these modules independently they are included in the Ansible Azure package.

* [azure_rm_resourcegroup](https://docs.ansible.com/ansible/latest/modules/azure_rm_resourcegroup_module.html#azure-rm-resourcegroup-module)
* [azure_rm_virtualnetwork](https://docs.ansible.com/ansible/latest/modules/azure_rm_virtualnetwork_module.html#azure-rm-virtualnetwork-module)
* [azure_rm_subnet](https://docs.ansible.com/ansible/latest/modules/azure_rm_subnet_module.html#azure-rm-subnet-module)
* [azure_rm_publicipaddress](https://docs.ansible.com/ansible/latest/modules/azure_rm_publicipaddress_module.html#azure-rm-publicipaddress-module)
* [azure_rm_securitygroup](https://docs.ansible.com/ansible/latest/modules/azure_rm_securitygroup_module.html#azure-rm-securitygroup-module)
* [azure_rm_networkinterface](https://docs.ansible.com/ansible/latest/modules/azure_rm_networkinterface_module.html#azure-rm-networkinterface-module)
* [azure_rm_virtualmachine](https://docs.ansible.com/ansible/latest/modules/azure_rm_virtualmachine_module.html#azure-rm-virtualmachine-module)

## Step 1 - Create a Resource Group <a name="step-1-create-a-resource-group"</a>

Azure resource groups are used to logically group related resources. Resource groups help organize your cloud environment and can also be used to grant access to specific workloads within the resource group. They are also required when creating other Azure resources. Such as virtual networks. Another benefit of resource groups is easy cleanup. When you delete a resource group everything inside of it is deleted with it. 

In order to create an Azure resource group with Ansible use the `azure_rm_resourcegroup` module. It requires two parameters; `name` and `location`. The name parameter will become the name of the resource group and the location is the Azure region the resource group is placed in. Placing a resource group in one region does not mean you can only use that region for other Azure resources inside the resource group. It is simply where the metadata of the resource group is located.

```yaml
- name: Create resource group
    azure_rm_resourcegroup:
    name: rg-cs-ansible
    location: eastus
```

## Step 2 - Create a Virtual Network <a name="step-2-create-a-virtual-network"</a>

Virtual networks allow you to build out your private network within Azure. Virtual networks are used to connect resources running within an Azure data center. Even though they are virtual and exist out of reach to you. The same networking principals apply. The virtual network requires two main parts; address space and a subnet or subnets. 

In order to deploy the virtual network in Azure with Ansible you'll need use two Ansible modules; `azure_rm_virtualnetwork` and `azure_rm_subnet`. The subnet module depends on the network module and for that reason the network module will be defined first. Ansible is procedural and the order of the tasks is very important. 

The task `Create virtual network` requires three parameters; `resource_group`, `name`, and `address_prefixes`. The resource_group is the name given to the previous tasks which created the resource group. `vnet-cs-web` is the name given to the virtual network resource and will be used when creating the subnet. Address prefixes of `10.0.0.0/16` defines the address space of the virtual network.

```yaml
- name: Create virtual network
    azure_rm_virtualnetwork:
    resource_group: rg-cs-ansible
    name: vnet-cs-web
    address_prefixes: "10.0.0.0/16"
```

`Add subnet` is the next tasks which is used to add a subnet range to the virtual network. A subnet is a logical subdivision of an IP network and in this example it is used to carve out a section of IP addresses for web machines within that network. The task uses the `azure_rm_subnet` Ansible module. The subnet is given the name of `snet-cs-web` followed by the address prefix of the subnet `10.0.1.0/24` and it specifies the virtual network where the subnet will be created which is `vnet-cs-web`.  

```yaml
- name: Add subnet
    azure_rm_subnet:
    resource_group: rg-cs-ansible
    name: snet-cs-web
    address_prefix: "10.0.1.0/24"
    virtual_network: vnet-cs-web
```

## Step 3 - Create a Public Ip <a name="step-3-create-a-public-ip"</a>

While the virtual network created previously will assign a private Ip address to the virtual machine the Public Ip address assigns a public Ip address to the virtual machine. Without a Public Ip address you won't be able to communicate with the virtual machine without a VPN or other means. To create an Azure Public Ip address resource using Ansible, you'll use the `azure_rm_publicipaddress` module.

In order to create an Azure Public Ip address you'll need to define the following parameters; resource group, allocation method, and name. Specifying the resource group of rg-cs-ansible places the public ip address in the same location as the rest of the resources already created. Allocation method determines when the Ip address is assigned to the resource and when the Ip address is released. There are two options for allocation method, static and dynamic and the resource SKU determines which of these you can choose. To learn more about these options check out [IP address types and allocation methods in Azure](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-ip-addresses-overview-arm#sku). The name of the resource is `pip-cs-web`.  

Perhaps you want to return the public Ip address assigned right after the resource is created. You can accomplish this by using an Ansible command called [register](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#registering-variables_). Register allows you to populate variables based on the output of Ansible tasks. In this example the output of the task Create public IP address is assigned to a variable named `output_ip_address_`. Registering the variable does not output the information. Which is what the debug command does within the `Output public IP` task. Because output from tasks are in JSON you can parse the results to display what you want. In this example the variable output_ip_address is parsed to display only the ip address and nothing else.

```yaml
- name: Create public IP address
    azure_rm_publicipaddress:
    resource_group: rg-cs-ansible
    allocation_method: Static
    name: pip-cs-web
    register: output_ip_address

- name: Output public IP
    debug:
    msg: "The public IP is {{ output_ip_address.state.ip_address }}"
```

## Step 4 - Create a Network Security Group <a name="step-4-create-a-network-security-group"</a>

Azure Network Security Groups are what filter network traffic based on a set of rules that are defined. You can think of them as being similar to a network firewall. Without a network security group allowing traffic in, you will not be able to connect to your virtual machine within Azure. For that reason it is necessary to use the `azure_rm_securitygroup` module to create one and also define some rules allowing traffic in.

In this example you'll want to connect to the virtual machine using three different protocols; RDP, HTTP, HTTPS, and WimRM. Each of these uses different inbound ports to communicate. RDP uses 3389, HTTP uses 80, HTTPS uses 443, and WinRM uses 5985 and 5986. Each of these protocols will require a set of rules associated with the network security group allowing inbound traffic in. To create an Azure network security group you only need two pieces of information. The resource group to associate it with and the name of the network security group. However, you'll need more information to create rules associated with it.

It is a common practice to create one rule per protocol. In this example, you'll create three rules. When creating the rules each one requires you specify a name, protocol, destination port range, access, priority, and direction.

```yaml
- name: Create Network Security Group
    azure_rm_securitygroup:
    resource_group: rg-cs-ansible
    name: nsg-cs-web
    rules:
        - name: 'allow_rdp'
        protocol: Tcp
        destination_port_range: 3389
        access: Allow
        priority: 1001
        direction: Inbound
        - name: 'allow_web_traffic'
        protocol: Tcp
        destination_port_range:
            - 80
            - 443
        access: Allow
        priority: 1002
        direction: Inbound
        - name: 'allow_powershell_remoting'
        protocol: Tcp
        destination_port_range:
            - 5985
            - 5986
```

## Step 5 - Create a Virtual Network Interface Card <a name="step-5-create-a-virtual-network-interface-card"</a>

Every computer virtual or not requires some form of network interface card to communicate with computers outside of itself. That is where the Azure virtual network interface card comes in. Previous to this you've created several virtual networking components; a virtual network, subnet,  a public ip address, and a network security group. All of these resources will be assigned to a virtual network interface card giving the Azure virtual machine private and public network access.

The Ansible module to create an Azure virtual network interface is `azure_rm_networkinterface`. In order to properly configure the interface you must associate it with a virtual network, subnet, network security group, and to access the virtual machine from the internet a public Ip address.

```yaml
- name: Create a network interface
    azure_rm_networkinterface:
    name: nic-cs-web
    resource_group: rg-cs-ansible
    virtual_network: vnet-cs-web
    subnet_name: snet-cs-web
    security_group: nsg-cs-web
    ip_configurations:
        - name: default
        public_ip_address_name: pip-cs-web
        primary: True
```

## Step 6 - Create a Virtual Machine <a name="step-6-create-a-virtual-machine"</a>

Everything to this point as prepared the Azure environment for you to deploy the virtual machine. You have a lot of options when it comes to deploying a virtual machine in Azure. However, in this tutorial you'll be deploying a Standard DS1 v2 Windows Server 2019 virtual machine. The Ansible module used to deploy Azure virtual machines is `azure_rm_virtualmachine`. 

As with every Azure resource you've created so far the virtual machine requires a resource group and a name. The name will be used for the Azure resource and the virtual machine's hostname. The `vm_size` uses Azure a Standard_DS1_v2 instance. You can learn more about Azure's instance sizes [here](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes). Windows virtual machines also require a user name and password be specified when you create the virtual machine. `admin_username` is set to azureuser and will be the local user account you'll login with. `admin_password` is set to `"{{ password }}"` which is an Ansible variable. To avoid having the password stored in clear text, you'll prompt for that input in the next section.

`nic-cs-web` is the only network interface you'll need to define at this time. It will connect the virtual machine to the private and public Ip addresses you defined when creating the network resources. `os_type` defines which Operating System will be used by the virtual machine. Setting this to `Windows` changes the behavior of the default parameters passed in to the task. `image` has several sub parameters that define the image used to build the virtual machine. When choosing an image you must specify; the offer (WindowsServer), the publisher (MicrosoftWindowsServer), sku (2019-Datacenter), and version (latest).

```yaml
- name: Create VM
    azure_rm_virtualmachine:
    resource_group: rg-cs-ansible
    name: vm-cs-web01
    vm_size: Standard_DS1_v2
    admin_username: azureuser
    admin_password: "{{ password }}"
    network_interfaces: nic-cs-web
    os_type: Windows
    image:
        offer: WindowsServer
        publisher: MicrosoftWindowsServer
        sku: 2019-Datacenter
        version: latest
```

## Step 7 - Deploy an Azure Windows Virtual Machine <a name="step-7-deploy-an-azure-windows-virtual-machine"</a>

Throughout this tutorial you've seen snippets of Ansible tasks. When putting those tasks into a playbook you have to add a few additional sections. `hosts` is used to define which target the playbook will be executed against. Setting this to localhost will run the playbook on the Ansible server itself.

`var_prompts` is used to prompt for Ansible variables when the playbook is executed. It's a simple choice for populating variables that contain sensitive information that you do not want to store in the playbook file. You'll use it to prompt for and populate the password used by the task creating the virtual machine.

`tasks` define the sequential tasks that are executed to complete the playbook. The order of these tasks is very important. For example, the task creating the resource group must be at the top. Without the resource group all the tasks after it will fail because the resource group does not exist yet.

Because the tasks used in this playbook create Azure resources a connection from Ansible to Azure must be establish before you can execute the playbook. You have several options available to connect Ansible to Azure. You can define environment variables or create an Ansible credential file. Both options are explained in dept in [Connecting to Azure with Ansible](https://dev.to/joshduffney/connecting-to-azure-with-ansible-22g2).

{% link https://dev.to/joshduffney/connecting-to-azure-with-ansible-22g2 %}

```yaml
#deployWindowsAzureVirtualMachine.yaml
---
- hosts: localhost
  connection: local

  vars_prompt:
    - name: password
      prompt: "Enter local administrator password"

  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: rg-cs-ansible
        location: eastus

    - name: Create virtual network
      azure_rm_virtualnetwork:
        resource_group: rg-cs-ansible
        name: vnet-cs-web
        address_prefixes: "10.0.0.0/16"

    - name: Add subnet
      azure_rm_subnet:
        resource_group: rg-cs-ansible
        name: snet-cs-web
        address_prefix: "10.0.1.0/24"
        virtual_network: vnet-cs-web

    - name: Create public IP address
      azure_rm_publicipaddress:
        resource_group: rg-cs-ansible
        allocation_method: Static
        name: pip-cs-web
      register: output_ip_address

    - name: Output public IP
      debug:
        msg: "The public IP is {{ output_ip_address.state.ip_address }}"

    - name: Create Network Security Group
      azure_rm_securitygroup:
        resource_group: rg-cs-ansible
        name: nsg-cs-web
        rules:
          - name: 'allow_rdp'
            protocol: Tcp
            destination_port_range: 3389
            access: Allow
            priority: 1001
            direction: Inbound
          - name: 'allow_web_traffic'
            protocol: Tcp
            destination_port_range:
              - 80
              - 443
            access: Allow
            priority: 1002
            direction: Inbound
          - name: 'allow_powershell_remoting'
            protocol: Tcp
            destination_port_range:
              - 5985
              - 5986
            access: Allow
            priority: 1003
            direction: Inbound

    - name: Create a network interface
      azure_rm_networkinterface:
        name: nic-cs-web
        resource_group: rg-cs-ansible
        virtual_network: vnet-cs-web
        subnet_name: snet-cs-web
        security_group: nsg-cs-web
        ip_configurations:
          - name: default
            public_ip_address_name: pip-cs-web
            primary: True


    - name: Create VM
      azure_rm_virtualmachine:
        resource_group: rg-cs-ansible
        name: vm-cs-web01
        vm_size: Standard_DS1_v2
        admin_username: azureuser
        admin_password: "{{ password }}"
        network_interfaces: nic-cs-web
        os_type: Windows
        image:
          offer: WindowsServer
          publisher: MicrosoftWindowsServer
          sku: 2019-Datacenter
          version: latest
```

`ansible-playbook` is the Ansible command used to execute playbooks. Because the playbook is targeting localhost, the Ansible server itself no inventory or host file is required. All that is required is the name of the playbook to be executed. Save the above playbook as `deployWindowsAzureVirtualMachine.yaml` and run the following command to deploy your Windows virtual machine to Azure!

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/w5pwly78ga7glaamkrop.gif)

```yaml
ansible-playbook deployWindowsAzureVirtualMachine.yaml
```

### Conclusion <a name="conclusion"</a>

You've now deployed a Windows virtual machine to Azure! At this point you might be wondering what the benefit of using Ansible is over creating the resources in the Azure portal or by using a PowerShell script. The benefits of using Ansible are;

* Codified Infrastructure
* Idempotent Automation
* Configuration Management

If you had used the portal you'd have a difficult time recreating the environment exactly the way it was before. If you had scripted the creation of all these resources you wouldn't be able to run that same script over and over without a lot of modification and error handling. That is where the idempotent nature of Ansible is extremely valuable.
