FROM centos:centos7.7.1908

RUN  yum check-update; \
   yum install -y gcc libffi-devel python-devel openssl-devel epel-release; \
   yum install -y python3; \
   yum install -y python-pip python-wheel; \
   curl https://packages.microsoft.com/config/rhel/7/prod.repo | tee /etc/yum.repos.d/microsoft.repo; \
   yum install -y powershell-preview; \
   pwsh-preview -c "Set-PackageSource -Name PSGallery -Trusted"; \
   pwsh-preview -c "Install-Module AZ -Scope AllUsers"; \
   pip install --upgrade pip; 

RUN  pip install ansible[azure]; \
     pip install pywinrm;
     
RUN pip2.7 install cryptography==2.2.2
