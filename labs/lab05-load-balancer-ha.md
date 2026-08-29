# Lab 05 - Azure Load Balancing / High Availability

I completed Lab 05 by creating two Ubuntu Web Servers, placing them behind an Azure Load Balancer (Standard SKU) across different Availability Zones, and verifying High Availability through three failure tests.

This was my first lab working with **High Availability** and **traffic distribution**.

---

## Architecture overview

```text
Internet
   ↓
Load Balancer (Standard SKU, Static Public IP)
   ├── Health Probe (TCP 80, interval 5s, threshold 2)
   │
   ├── web01 (Zone 1, nginx)
   └── web02 (Zone 2, nginx)
```

Key design decisions:
- Web VMs have **no Public IP** — only the Load Balancer is reachable from the Internet.
- VMs are placed in **different Availability Zones** for DC-level isolation.
- The Health Probe is attached to the LB rule (without it, the LB cannot detect application-layer failures).

---

## Problems I encountered

### 1. Cloud-init file creation with `code`

I initially tried:

```bash
code ~/cloud-init-web.yaml
```

in Azure Cloud Shell, but the file could not be saved and showed:

```text
Unable to save cloud-init-web.yaml
```

### Solution

I used `nano` instead:

```bash
nano ~/cloud-init-web.yaml
```

This worked properly.

**Lesson:** For simple configuration files in Azure Cloud Shell, use `nano` when the `code` editor has problems.

---

### 2. Azure CLI Backend Pool command error

I initially used:

```bash
az network lb backend-pool address add
```

but my Azure CLI returned:

```text
'backend-pool' is misspelled or not recognized by the system.
```

I checked the available syntax with `--help` and used:

```bash
az network lb address-pool address add
```

instead.

**Lesson:** Azure CLI command syntax can differ depending on the installed CLI version/command group. When a command is not recognized, check:

```bash
az <command> --help
```

instead of guessing.

---

### 3. Backend Pool command was accidentally corrupted

While entering the long multi-line command, part of the command became corrupted, for example:

```text
--nic $WEB02_NIC_IDame web02-ip \add \
```

This caused Azure CLI to complain about missing arguments such as:

```text
-n/--name
--ip-address
```

### Solution

I stopped and rebuilt the command correctly, then executed the commands separately.

**Lesson:** Be careful with `\` line continuation and copy/paste when using long Azure CLI commands.

---

### 4. Cloud Shell session disconnected and temporary variables disappeared

The Cloud Shell disconnected during the lab.

Variables such as:

```bash
$RG
$LB_IP
$WEB01_NAME
```

were temporary Shell variables, so they disappeared when the session ended.

The Azure resources themselves were NOT deleted.

### Solution

I reconnected to Cloud Shell and recreated/retrieved the variables from Azure.

### Future improvement

From now on, I want to **export/save my lab variables into a file** so that I can restore them after reconnecting.

For example:

```bash
export RG="rg-lab05"
export LOCATION="northcentralus"
export VNET_NAME="vnet-lab05"
export SUBNET_NAME="snet-web"
export NSG_NAME="nsg-web"
export VM_SIZE="Standard_B2ats_v2"
export ADMIN_USER="azureuser"
export WEB01_NAME="web01"
export WEB02_NAME="web02"
export LB_NAME="lb-web"
export LB_PIP_NAME="pip-lb-web"
```

Then I can save them in a file such as:

```text
lab05-vars.sh
```

and restore them after reconnecting with:

```bash
source lab05-vars.sh
```

**Important distinction:**

* Azure resources → persistent in Azure
* Shell variables → temporary unless saved/exported

---

## Lab 05 technical lessons

### Load Balancer

The main purpose of the Load Balancer is to:

1. Distribute traffic across multiple VMs.
2. Improve High Availability (HA).

Example:

```text
Internet
   ↓
Load Balancer
   ├── web01
   └── web02
```

### Health Probe

The Health Probe checks whether the backend VM is healthy.

In this lab:

```text
Protocol: TCP
Port: 80
Interval: 5 seconds
Threshold: 2 (consecutive failures)
```

If the VM fails the health check **twice consecutively**, it is considered unhealthy.

The Load Balancer then stops sending new traffic to that unhealthy VM.

**Important:** Unhealthy does NOT mean the VM is stopped. The VM can still be running — only the application (nginx) is unresponsive.

---

## Health Probe failure test

I stopped nginx on web01:

```bash
sudo systemctl stop nginx
```

Then:

```text
web01 → Unhealthy
web02 → Healthy
```

The Load Balancer sent traffic only to web02.

After restarting nginx:

```bash
sudo systemctl start nginx
```

web01 became healthy again, and the Load Balancer started sending traffic to both VMs.

I confirmed this with:

```bash
for i in 1 2 3 4 5; do
  curl -s http://$LB_IP | grep -o 'Hello from [^<]*'
done
```

Example result:

```text
Hello from web02
Hello from web02
Hello from web01
Hello from web02
Hello from web01
```

This confirmed that the Load Balancer was distributing traffic between the two web servers.

---

## Important concepts learned

### VNet

Private network boundary in Azure.

### Subnet

A segment inside the VNet.

### NSG

Controls network traffic using rules such as allowing TCP port 80 and 22.

### VM

The actual web server.

### nginx

The web server software running inside the VM.

### Backend Pool

The group of VMs that can receive traffic from the Load Balancer.

### Health Probe

Checks whether a backend VM is healthy.

### Load Balancer

Distributes traffic and avoids unhealthy backend VMs.

### HA (High Availability)

Keeping a service available even when one server fails.

---

## Key troubleshooting workflow for future labs

When something fails:

1. Read the exact error message.
2. Check whether the command syntax is correct.
3. Use `--help` when necessary.
4. Check Azure resource status.
5. Check whether Shell variables still exist.
6. Recreate/retrieve variables if Cloud Shell was disconnected.
7. Test one component at a time.
8. Confirm the result before moving to the next step.

---

## Main takeaway

The most important thing I learned from Lab 05 was not just memorizing Azure CLI commands.

I learned how:

```text
Load Balancer
      ↓
Health Probe
      ↓
Healthy / Unhealthy
      ↓
Traffic distribution
      ↓
High Availability
```

works in practice.

I also learned that Azure resources and Cloud Shell variables are different things, and I should save/export my lab variables so I can restore my working environment after a Cloud Shell disconnection.