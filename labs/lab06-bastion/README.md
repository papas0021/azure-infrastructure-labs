# Lab 06 - Azure Bastion / Secure Private VM Access

I completed Lab 06 by creating a private Ubuntu VM without a Public IP and accessing it securely through Azure Bastion.

This lab focused on **secure remote access, private networking, and troubleshooting SSH connectivity**.

---

## Architecture overview

```text
Internet
   ↓
Azure Bastion
   ↓
AzureBastionSubnet
   ↓
Private IP: 10.0.1.4
   ↓
lab06-vm
(Ubuntu / SSH)
```

Key design decisions:

* The VM has **no Public IP**.
* Azure Bastion provides secure SSH access from the Azure Portal.
* The VM is placed in a private subnet.
* Bastion uses the dedicated `AzureBastionSubnet`.
* SSH access to the VM is controlled by an NSG.

---

## Resources

| Resource            | Configuration        |
| ------------------- | -------------------- |
| Resource Group      | `lab06-rg`           |
| Location            | `northcentralus`     |
| VNet                | `lab06-vnet`         |
| VM Subnet           | `vm-subnet`          |
| VM Subnet CIDR      | `10.0.1.0/24`        |
| Bastion Subnet      | `AzureBastionSubnet` |
| Bastion Subnet CIDR | `10.0.2.0/24`        |
| VM                  | `lab06-vm`           |
| VM Private IP       | `10.0.1.4`           |
| VM Public IP        | None                 |
| VM Size             | `Standard_B2ats_v2`  |
| Bastion             | `lab06-bastion`      |
| Bastion SKU         | Basic                |
| SSH Port            | TCP 22               |

---

## 1. Create the Virtual Network

I created a VNet with two separate subnets:

```text
lab06-vnet
├── vm-subnet
│   └── 10.0.1.0/24
│
└── AzureBastionSubnet
    └── 10.0.2.0/24
```

The `AzureBastionSubnet` name is required by Azure Bastion and must be used exactly.

---

## 2. Create a Private VM

The VM was created without a Public IP:

```bash
az vm create \
  --resource-group $RG \
  --name $VM \
  --image Ubuntu2204 \
  --size Standard_B2ats_v2 \
  --vnet-name $VNET \
  --subnet vm-subnet \
  --public-ip-address "" \
  --generate-ssh-keys
```

The VM received a private IP:

```text
10.0.1.4
```

There was no Public IP assigned to the VM.

### Why no Public IP?

A VM with a Public IP can be directly exposed to the Internet.

For example:

```text
Internet
   ↓
Public IP
   ↓
VM
```

This can increase the attack surface.

With Bastion:

```text
Internet
   ↓
Bastion
   ↓
Private VM
```

The VM itself does not need to be directly exposed to the Internet.

---

## 3. Create Azure Bastion

I created an Azure Bastion host using the dedicated Bastion subnet.

The Bastion resource was successfully provisioned.

```text
lab06-bastion
        ↓
AzureBastionSubnet
        ↓
Private VM
```

The Bastion host provides browser-based SSH access to the private VM.

---

## 4. Configure Network Security

The VM NIC was associated with:

```text
lab06-vmNSG
```

The NSG contained an inbound SSH rule:

```text
Priority: 1000
Access: Allow
Protocol: TCP
Destination Port: 22
Direction: Inbound
```

This allows SSH traffic to reach the VM.

---

## 5. Connect to the VM through Bastion

I initially experienced a Bastion connection error.

The VM itself was running:

```text
VM State: Running
```

The Bastion host was also healthy:

```text
Provisioning State: Succeeded
```

The VM had:

```text
Private IP: 10.0.1.4
Public IP: None
```

---

# Problems I encountered

## 1. SSH key was not supplied during VM creation

My first VM creation attempt failed because Azure required an SSH key.

The error indicated that an RSA key file or key value needed to be supplied.

### Solution

I recreated the command using:

```bash
--generate-ssh-keys
```

Azure generated the SSH key pair in Cloud Shell:

```text
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

The private key was then used by Azure Bastion for authentication.

---

## 2. Bastion connection error

When I first tried to connect through Bastion, the connection failed with a timeout/connection error.

I checked the Bastion status:

```bash
az network bastion show \
  --resource-group $RG \
  --name lab06-bastion \
  --query provisioningState
```

Result:

```text
Succeeded
```

I also checked the VM:

```bash
az vm get-instance-view \
  --resource-group $RG \
  --name $VM
```

The VM was running.

At this point, I knew that the problem was probably not simply that the VM or Bastion was stopped.

---

## 3. Check the VM SSH service

I used Azure VM Run Command to execute a command directly inside the VM:

```bash
az vm run-command invoke \
  --resource-group $RG \
  --name $VM \
  --command-id RunShellScript \
  --scripts "systemctl status ssh --no-pager"
```

The SSH service was:

```text
Active: active (running)
```

It was also listening on port 22.

I confirmed this with:

```bash
az vm run-command invoke \
  --resource-group $RG \
  --name $VM \
  --command-id RunShellScript \
  --scripts "sudo ss -tlnp | grep ':22'"
```

Result:

```text
LISTEN 0 128 0.0.0.0:22
LISTEN 0 128 [::]:22
```

This confirmed that the SSH server was running and listening on port 22.

---

## 4. Root cause: Incorrect username capitalization

The most important clue appeared in the SSH logs:

```text
Invalid user Sasuga from 10.0.2.4
```

This showed that Bastion was actually reaching the VM.

The problem was the username:

```text
Sasuga
```

instead of:

```text
sasuga
```

Linux usernames are case-sensitive.

Therefore:

```text
Sasuga ≠ sasuga
```

### Solution

I changed the Bastion username to:

```text
sasuga
```

and used the correct SSH private key.

The Bastion connection then worked successfully.

---

# Troubleshooting workflow

This was the troubleshooting process I used:

```text
Bastion connection error
        ↓
Check Bastion status
        ↓
Check VM status
        ↓
Check VM Private IP
        ↓
Check NSG
        ↓
Check SSH service
        ↓
Check port 22
        ↓
Check SSH logs
        ↓
"Invalid user Sasuga"
        ↓
Username case mismatch
        ↓
Change "Sasuga" → "sasuga"
        ↓
Connection successful
```

This was an important lesson in troubleshooting: **do not assume that a connection error means the network itself is broken.**

---

# VM Run Command

One of the most useful tools in this lab was:

```bash
az vm run-command invoke
```

This allows commands to be executed inside a VM through the Azure management plane without requiring an SSH login.

For example:

```bash
az vm run-command invoke \
  --resource-group $RG \
  --name $VM \
  --command-id RunShellScript \
  --scripts "systemctl status ssh --no-pager"
```

This was especially useful because the SSH connection itself was failing.

Instead of needing SSH to troubleshoot SSH, I could use Azure Run Command to inspect the VM from outside.

### Important limitation

VM Run Command is intended for script/command execution rather than interactive shell sessions.

Sensitive information should also be handled carefully when passing values through commands or scripts.

---

# VM Internal Checks

After connecting through Bastion, I checked the VM from inside.

### Check network configuration

```bash
ip addr
```

This confirmed the VM's private IP address:

```text
10.0.1.4
```

### Check hostname

```bash
hostname
```

The hostname was:

```text
lab06-vm
```

### Internet connectivity test

I also tested:

```bash
curl ifconfig.me
```

The command did not return a result and had to be cancelled with `Ctrl+C`.

This showed that outbound Internet connectivity was not available through this configuration.

However, this did not prevent the main purpose of the lab from working because the objective was to access the private VM through Azure Bastion.

---

# Important Security Concepts

## Public IP on a VM

Giving a VM a Public IP can increase its exposure to the Internet.

Potential risks include:

1. **Brute-force attacks**
2. **Port scanning and service exposure**
3. **Direct exploitation if credentials or SSH keys are compromised**

For this reason, keeping the VM private and using Bastion can reduce its direct Internet exposure.

---

## Azure Bastion

Azure Bastion provides secure remote access to VMs through Azure without requiring the VM itself to have a Public IP.

The basic architecture is:

```text
User
  ↓
Internet
  ↓
Azure Bastion
  ↓
Private VM
```

This is useful when administrators need remote access to private infrastructure.

---

## AzureBastionSubnet

The Bastion subnet must use the reserved name:

```text
AzureBastionSubnet
```

Azure expects this specific subnet name for Bastion deployment.

---

## NSG

A Network Security Group controls network traffic using rules.

In this lab, TCP port 22 was allowed for SSH access.

```text
Bastion
   ↓
TCP 22
   ↓
VM
```

The Bastion-to-VM connection was therefore able to reach the SSH service.

---

# Public IP SKU and Static Allocation

The Bastion requires a Public IP so that users can reach the Bastion service from the Internet.

The Public IP can be configured with a SKU and allocation method.

Example:

```bash
--sku Standard \
--allocation-method Static
```

### Standard SKU

The SKU determines the service level, supported features, and limitations of the Public IP resource.

### Static allocation

```text
--allocation-method Static
```

means the Public IP address is intended to remain assigned rather than being dynamically reassigned.

These are separate settings:

```text
Public IP
├── SKU: Standard
└── Allocation: Static
```

---

# Bastion vs Cloud Shell VNet Integration

Both can provide ways to work with resources in Azure networking, but their use cases are different.

### Cloud Shell VNet Integration

Useful when running commands or tools from a shell environment that needs connectivity to resources inside a VNet.

### Azure Bastion

Useful when a user needs interactive SSH or RDP access to a private VM through the Azure Portal.

In this lab, Bastion was used because the goal was:

```text
External user
      ↓
Bastion
      ↓
Private VM
```

---

# Key Commands

### Check VM IP

```bash
az vm list-ip-addresses \
  --resource-group $RG \
  --name $VM \
  --output table
```

### Check Bastion status

```bash
az network bastion show \
  --resource-group $RG \
  --name lab06-bastion \
  --query provisioningState
```

### Check SSH service

```bash
az vm run-command invoke \
  --resource-group $RG \
  --name $VM \
  --command-id RunShellScript \
  --scripts "systemctl status ssh --no-pager"
```

### Check port 22

```bash
az vm run-command invoke \
  --resource-group $RG \
  --name $VM \
  --command-id RunShellScript \
  --scripts "sudo ss -tlnp | grep ':22'"
```

---

# Cleanup

After completing the lab, I deleted the entire resource group to avoid unnecessary Azure charges:

```bash
az group delete \
  --name lab06-rg \
  --yes
```

This removed the resources created for the lab, including the VM, VNet, NSG, Bastion, and Public IP.

---

# Important concepts learned

### Private VM

A VM without a Public IP is not directly exposed to the Internet.

### Azure Bastion

Provides secure remote access to private VMs without requiring a Public IP on the VM.

### AzureBastionSubnet

A dedicated subnet required for Azure Bastion.

### NSG

Controls inbound and outbound network traffic using security rules.

### SSH

Secure remote administration protocol commonly used for Linux VMs.

### VM Run Command

Allows commands to be executed inside an Azure VM without requiring an interactive SSH connection.

### Private IP

Used for communication within the VNet and connected networks.

### Public IP

Provides Internet-reachable addressing for Azure resources that require it.

---

# Key Troubleshooting Lessons

When Bastion or SSH does not work:

1. Check the Bastion provisioning state.
2. Check whether the VM is running.
3. Check the VM's Private IP.
4. Check the NSG rules.
5. Check whether SSH is running.
6. Check whether port 22 is listening.
7. Check the SSH logs.
8. Check the username and authentication method.
9. Do not assume a connection error is a network problem.
10. Use `az vm run-command invoke` when normal SSH access is unavailable.

The most important troubleshooting lesson from this lab was:

> **Follow the evidence instead of guessing.**

The Bastion connection initially looked like a networking problem, but the VM logs showed that Bastion was already reaching the SSH service.

The actual problem was simply the case-sensitive username:

```text
Sasuga ❌
sasuga  ✅
```

---

# Main takeaway

The main thing I learned from Lab 06 was how to securely access a private Azure VM without giving the VM a Public IP.

The overall architecture was:

```text
Internet
    ↓
Azure Bastion
    ↓
AzureBastionSubnet
    ↓
Private IP
    ↓
Linux VM
    ↓
SSH
```

I also learned a practical troubleshooting workflow for Azure networking and SSH.

The most valuable experience was using:

```bash
az vm run-command invoke
```

to investigate a VM when SSH access was not working.

The command allowed me to check the SSH service and logs directly inside the VM and eventually identify the real problem:

```text
Invalid user Sasuga
```

After changing the username to lowercase `sasuga`, Bastion connected successfully.

This lab helped me understand the difference between **network connectivity problems, service problems, and authentication problems**, rather than treating every connection failure as a network issue.
