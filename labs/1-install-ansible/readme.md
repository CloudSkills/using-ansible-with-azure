# Install Ansible

## [Install Ansible on Linux virtual machines in Azure](https://docs.microsoft.com/en-us/azure/ansible/ansible-install-configure?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fansible%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json)

### Step 1 - Create a Resource Group

```powershell
New-AzResourceGroup -Name rg-cs-ansible -Location eastus
```

### Step 2 - Create a Linux VM with an Azure Resource Manager Template

```powershell
#update azuredeploy.parameters.json  Admin Password value
New-AzResourceGroupDeployment -ResourceGroupName 'rg-cs-ansible'  -TemplateFile ./azuredeploy.json -TemplateParameterFile ./azuredeploy.parameters.json
```

### Step 3 - Install Ansible on Linux virtual machine in Azure

1. SSH to Azure Linux VM

    ```powershell
    (Get-AzPublicIpAddress -ResourceGroupName 'rg-cs-ansible' -Name 'pip-dev-ansible').IpAddress
    ssh sysadmin@<PublicIpAddress>
    ```

2. Install the required packages for the Azure Python SDK modules

    ```bash
    sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
    ```

3. Install the required packages Ansible

    ```bash
    sudo pip install ansible[azure]
    ```

### Step 4 - Run Ansible Commands

```bash
#moduleName HostGroup Module ShellCommand

#run a shell command
ansible localhost -m shell -a 'date'

#run a ping test
ansible localhost -m ping

#create an Azure resource group
ansible localhost -m azure_rm_resourcegroup -a 'name=rg-cs-ansible location=eastus'
```

## Use a Docker Container for Ansible Development

### Step 1 - Install Docker Desktop

Docker Desktop supports Windows and Mac.

1. Download [Docker Desktop](https://www.docker.com/products/docker-desktop).
2. Install Docker Desktop

### Step 2 - Build the Docker File

```PowerShell
Code Dockerfile
```

### Step 3 - Pull the Docker Image

```bash
docker pull duffney/ansibleopsbox:latest
```

### Step 4 - Run the Docker Container

```bash
#Change Directory Location to labs/1-install-ansible
docker run -it -w /sln -v "$(pwd)":/sln duffney/ansibleopsbox:latest
```

### Step 5 - Run Ansible Commands

```bash
#moduleName HostGroup Module ShellCommand

#run a shell command
ansible localhost -m shell -a 'date'

#run a ping test
ansible localhost -m ping

#create an Azure resource group
ansible localhost -m azure_rm_resourcegroup -a 'name=rg-cs-ansible location=eastus'
```

## Conclusion

Using an Azure virtual machine allows you to take advantage of managed identities in Azure bypassing the need to store credentials on the system in either a credential file or environment variables. Containers have the benefit of being light weight and disposable. Containers are a great way to provide a consistent development environment for your team while also given the individual team members the flexibility to modify the environment without effecting other team members.
