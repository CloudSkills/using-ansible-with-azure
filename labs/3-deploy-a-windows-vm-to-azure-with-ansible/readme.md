# Deploy a Windows VM to Azure with Ansible

## Step 1 - Create the playbook

```bash
vi deployWindowsAzureVirtualMachine.yaml
```

## Step 2 - Define the Hosts block

```yml
---
- hosts: localhost
  connection: local
```

## Step 3 - Add Tasks to Deploy an Windows VM to Azure

### Add the Tasks block

```yml
---
- hosts: localhost
  connection: local

  tasks:
```

### Create a Resource Group

```yml
---
- hosts: localhost
  connection: local

  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: rg-cs-ansible
        location: eastus
```

### Create a Virtual Network

```yml
- name: Create virtual network
    azure_rm_virtualnetwork:
    resource_group: rg-cs-ansible
    name: vnet-cs-web
    address_prefixes: "10.0.0.0/16"
```

### Add a Subnet to the Virtual Network

```yml
- name: Add subnet
    azure_rm_subnet:
    resource_group: rg-cs-ansible
    name: snet-cs-web
    address_prefix: "10.0.1.0/24"
    virtual_network: vnet-cs-web
```

### Create a Public Ip Address

```yml
    - name: Create public IP address
      azure_rm_publicipaddress:
        resource_group: rg-cs-ansible
        allocation_method: Static
        name: pip-cs-web
      register: output_ip_address
```

### Output Public Ip Address

```yml
- name: Output public IP
    debug:
    msg: "The public IP is {{ output_ip_address.state.ip_address }}"
```

### Create a Network Security Group

```yml
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
```

### Create a Network Interface

```yml
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

### Create a Virtual Machine

```yml
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

## Step 4 - Use vars_prompt for the Local Administrator Password

```yml
  vars_prompt:
    - name: password
      prompt: "Enter local administrator password"
```

## Step 5 - Run the Ansible Playbook

```bash
ansible-playbook deployWindowsAzureVirtualMachine.yaml
```
