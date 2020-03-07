# Configure a Windows Web Server with Ansible in Azure

## Step 1 - Create the playbook

1. Create the file

    ```bash
    vi configureWindowsWebServer.yaml
    ```

2. Create the hosts block

    ```yml
    ---
    - hosts: all
    ```

## Step 2 - Set Ansible Connection Variables

```yaml
vars:
  ansible_user: azureuser
  ansible_password: '<PASSWORD>'
  ansible_connection: winrm
  ansible_winrm_transport: ntlm
  ansible_winrm_server_cert_validation: ignore
```

## Step 3 - Installing IIS with win_feature

```yaml
  - name: Install IIS
    win_feature:
        name: web-server
        include_management_tools: yes
        include_sub_features: yes
        state: present
```

## Step 4 - Set the Index Page with win_copy

1. Create index.html

    ```bash
    vi index.html
    ```

2. Add html contents to index.html

    ```html
    <!DOCTYPE html>
    <html>
    <body>
    <h1>Cloudskills: Using Ansible with Azure</h1>
    </body>
    </html>
    ```

3. Copy the index.html to the Windows Azure VM

    ```yml
    - name: Copy index.html to wwwroot
    win_copy:
      src: index.html
      dest: C:\inetpub\wwwroot\index.html
      force: yes
    ```

_The Ansible repository is the build artifact._

## Step 5 - Install DotNet Core Runtime with win_chocolatey

```yaml
  - name: install net core iis hosting module with no frameworks
    win_chocolatey:
      name: "dotnetcore-windowshosting"
      version: "3.1.0"
      install_args: "OPT_NO_RUNTIME=1 OPT_NO_SHAREDFX=1 OPT_NO_X86=1 OPT_NO_SHARED_CONFIG_CHECK=1"
      state: present
    notify: restart IIS
```

## Step 6 - Create a Logs Directory with win_file

```yaml
  - name: Create logging directory
    win_file:
        path: c:\logs
        state: directory
```

## Step 7 - Restart IIS with handlers

```yaml
  handlers:
    - name: restart IIS
      win_shell: '& {iisreset}'
```

## Step 8 - Configure the Windows Web Server with Ansible

1. Get Public Ip Address

    ```powershell
    (Get-AzPublicIpAddress -ResourceGroupName rg-cs-ansible -Name pip-cs-web).ipaddress
    ```

2. Run Ansible Playbook

    ```bash
    ansible-playbook configureWindowsWebServer.yaml -i <IpAddress>,
    ```

3. Curl Web Server

    ```bash
    curl <IpAddress>
    ```
