# Lab 04 - Azure VM / NSG / Nginx / HTTPS

I completed Lab 04 by creating an Azure Ubuntu VM, configuring an NSG, setting up Nginx, and enabling HTTPS with a self-signed certificate.

During the lab, I encountered several issues and worked through them step by step.

## 1. Resource Group issue

I initially used `export RG="lab04-rg-$(date +%s)"` but later accidentally used `lab04-rg` directly. I checked the actual Azure resource groups with `az group list -o table` and found the correct one: `lab04-rg-1787511541`.

I then updated the environment variable: `export RG="lab04-rg-1787511541"`

**Lesson**: Always verify environment variables before running Azure CLI commands.

## 2. SSH key issue

My Cloud Shell session temporarily disconnected, and the SSH private key in `/home/sasuga/.ssh/lab04_id_rsa` was lost. When I tried to use scp and ssh, I received: `Permission denied (publickey)`.

**Investigation**:
- Checked SSH public key on the VM with: `az vm show -g $RG -n $VM --query "osProfile.linuxConfiguration.ssh.publicKeys" -o json`
- Generated a new SSH key pair
- Confirmed via `ssh-keygen -y` that the new private key matched a different public key
- Added the new public key to the VM's `/home/azureuser/.ssh/authorized_keys`

**Result**: SSH and SCP worked again without recreating the VM.

## 3. Public IP / NSG issue

My public IP changed after reconnecting to Cloud Shell. I checked my current public IP with: `export MYIP=$(curl -s https://api.ipify.org)`

Then I updated the existing NSG SSH rule:
az network nsg rule update -g 
RG−−nsg−name
RG−−nsg−nameNSG -n allow-ssh-from-myip --source-address-prefixes $MYIP

Lesson: Understand the difference between VM public IP and my own client public IP.

NSG allowed:

TCP 22 from my public IP
TCP 80 from the Internet
TCP 443 from the Internet
4. Nginx configuration
I created an Nginx configuration file, copied it to the VM using scp, and enabled it using a symbolic link.

I tested the configuration with sudo nginx -t and reloaded Nginx with sudo systemctl reload nginx.

Result: syntax is ok / test is successful

5. HTTPS
I created a self-signed certificate using OpenSSL and configured Nginx to use TLS 1.2 and TLS 1.3.

I verified HTTPS locally from the VM using curl -sk https://127.0.0.1 and accessed https://64.236.163.92 from my browser.

The browser showed a certificate warning because the certificate was self-signed and the certificate name was lab04.local rather than the VM's public IP.

6. Small Bash issue
When I tried to use Lab 4 Re-run Success! inside an echo command, Bash returned: bash: !: event not found

Lesson: ! can trigger Bash history expansion.

What I learned
The most important part of this lab was not memorizing commands. I learned how to troubleshoot by identifying which layer was actually failing:

Azure Resource Group
VNet / Subnet
NSG
Public IP
SSH connectivity
SSH authentication / keys
Nginx
TLS / HTTPS
Web browser
When something failed, I tried to determine which layer was responsible before changing anything.

Key distinctions I now understand:

Azure networking vs. Linux networking
NSG access vs. SSH authentication
Public IP vs. private IP
Private SSH key vs. public SSH key
Nginx configuration vs. TLS certificate
HTTP port 80 vs. HTTPS port 443 LABEOF