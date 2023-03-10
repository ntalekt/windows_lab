# Lab for VMware Horizon

- [Lab for VMware Horizon](#lab-for-vmware-horizon)
  * [Overview](#overview)
    + [Prerequisites](#prerequisites)
    + [Packer](#packer)
      - [File definitions](#files)
    + [Terraform](#terraform)
      - [File definitions](#files-1)
    + [Ansible](#ansible)
      - [File definitions](#files-2)
  * [Process](#process)
    + [Image Creation (packer)](#image-creation--packer-)
    + [Deploy VMs (terraform)](#deploy-vms--terraform-)
    + [Configure VMs (ansible)](#configure-vms--ansible-)
  * [Horizon Connection Servers](#horizon-connection-servers)
  * [Credentials](#credentials)
  * [Servers](#servers)
  * [References](#references)

## Overview

The goal of this environment is for some VMware Horizon testing. Currently I have a vSphere environment with several Ubuntu servers but zero Windows.

### Prerequisites

* **Hardware**: a vSphere environment (i have a single ESXi host that hosts a vCenter)
* **Software**: all the software prerequisites are baked into this vagrant ubuntu desktop: [vagrant-ubuntu-desktop](https://github.com/ntalekt/vagrant-ubuntu-desktop)

> **Note**
> If you already have a linux machine and you'd like to install the prerequisites you can look in the following scripts in [vagrant-ubuntu-desktop](https://github.com/ntalekt/vagrant-ubuntu-desktop)
> * [common](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/apt_update.sh)
> * [ansible](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/ansible.sh)
> * [packer & terraform](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/ansible.sh)

### Packer

Packer creates a Windows Server 2022 vSphere template and ovf that has VMware tools, and some other basic applications installed. 

Packer uses `autounattend.xml` and `sysprep-autounattend.xml` to automate Windows Settings windows_lab/packer/configs.

  * It pulls Windows Server 2022 Datacenter Eval Edition (Desktop Experience) from Microsoft's site
  * Installs & configure OpenSSH Client & Server for remote connection
  * Installs VMware tools from ISO provided from the build ESX server

#### File definitions

  * `myvarfile.json` All those quality values that will be used
  * `WinServ2022.pkr.hcl` The main top quality with variable declares at the top, and the provisioner steps after.
  * `scripts/win-update.ps1` runs first: updates windows
  * `scripts/adjustments.ps1` runs second: tweaks windows, and installs some tools
  * `scripts/cleanup.ps1` cleans up windows after all the updates
  * `scripts/install-vmware-tools-from-iso.ps1` installs VMware tools so it's baked into the base image.

**Packer Provisioner Steps**
* Updating OS via Windows Update
* Doing some OS adjustments
  * Set Windows telemetry settings to minimum
  * Show file extensions by default (TODO: might not work?)
  * Install [Chocolatey](https://chocolatey.org/) - a Windows package manager
    * Install Microsoft Edge (Chromium)
    * Install Win32-OpenSSH-Server
    * Install PowerShell Core
    * Install 7-Zip
    * Install Notepad++
  * Enable Powershell-Core (`pwsh`) to be the default SSHD shell
* Cleanup tasks
* Remove CDROM drives from VM template (otherwise there would be 2)

### Terraform

Terraform deploys the required virtual machines against infrastructure using the vSphere provider in this case.

#### File definitions

* `variables.tf` declares the variables that will be used
* `terraform.tfvars` All those quality values that will be used
* `base.tf` defining the vSphere provider and common stuffs
* `01-PDF.tf` defining the Primary Domain Controller VM
* `01-ConnServ1.tf` defining the Connection Server 1 (standard)
* `02-ConnServ2.tf` defining the Connection Server 2 (replica1)
* `03-ConnServ3.tf` defining the Connection Server 3 (replica2)

### Ansible 

#### `winlab_install.yml`

Configures the VMs once they are deployed. 

* Setup Windows Server Feature: **Domain**
  * Primary Domain Controller
  * Auto-Join the Virtual Machines to the domain
  * Create a Horizon user and group within Active Directory
* Install Horizon Connection: Primary
  * TODO: Configure Events DB
* Install Horizon Connection: Replica
  * Register with Primary
* Common Configurations
  * Enable RDP and allow it through the firewall on all windows servers created

#### `connection_server_upgrade.yml`

Upgrade the VMware Connection Servers to a new version 

* Upgrade Horizon Connection: Primary/Replica
  * Check currently installed version
  * Take VMware snapshot
  * Take Connection Server backup
    * Located: C:\Install\Backup-*
  * Disable Connection Server client authentication
  * Download new Connection Server binary
  * Upgrade Connection Server
  * Check currently installed version
    * Enable Connection Server client authentication
    * Reboot

#### File definitions

* `inventory.yml` inventory of the hosts we will be touching
* `winlab_install.yml` an association of the roles to the servers for the greenfield base install/configure
* `connection_server_upgrade.yml` an association of the roles to the servers for the Connection Server upgrade
* `ansible.cfg` main hotness
* `group_vars/all.yml` defines all they key:value pairs needed
* `roles/*` all the different roles and stuff that tells ansible what it needs to do (**hint**: look in `winlab_install.yml` and `connection_server_upgrade.yml`)

## Process

### Image Creation (packer)

* Update variables in `myvarfile.json.example` and rename

    ```bash
    mv myvarfile.json.example myvarfile.json
    ```

* Update variables in `WinServ2022.pkr.hcl` (mainly the location of the VMware tools /ISO/windows.iso)

* Initialize packer

    ```bash
    packer init -upgrade WinServ2022.pkr.hcl
    ```

* Create the template

    ```bash
    packer build -timestamp-ui -force -var-file=myvarfile.json WinServ2022.pkr.hcl
    ```
> **Note**
> This will result in a template in your vSphere infrastructure named WinServ2022 and an OVF in the build directory.

```bash
2022-12-17T16:41:40-07:00: Build 'WinServ2022.vsphere-iso.WinServ2022' finished after 1 hour 11 minutes.

==> Wait completed after 1 hour 11 minutes

==> Builds finished. The artifacts of successful builds are:
--> WinServ2022.vsphere-iso.WinServ2022: WinServ2022
```

### Deploy VMs (terraform)

* Update variables in `terraform.tfvars.example` and rename

    ```bash
    mv terraform.tfvars.example terraform.tfvars
    ```

* Initialize terraform

    ```bash
    terraform init
    ```

* Terraform plan in order to detect an my errors

    ```bash
    terraform plan
    ```

* Actually deploy the VMs (parallelism=1 because my lab is slow :turtle:) 

    ```bash
    terraform apply -auto-approve -parallelism=1
    ```

> **Note**
> This will result in 4 VMs being creating using the variables defined in `terraform.tfvars`

```bash
...
vsphere_virtual_machine._PDC: Creation complete after 10m11s [id=420b41aa-e3fc-8ae7-19a2-537ba43fb62b]
...
vsphere_virtual_machine._ConnServ: Creation complete after 10m59s [id=420bf9b2-4ed7-a291-fe8f-df3a07d019ab]
...
vsphere_virtual_machine._ConnServ2: Creation complete after 9m50s [id=420baf8a-8b33-95d0-8720-b0efb2e56f1f]
...
vsphere_virtual_machine._ConnServ3: Creation complete after 7m38s [id=420b9a46-9743-c9ad-800e-90133a5a2084]
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```
> **Warning**
> Remove the VMs by running `terraform apply -auto-approve -destroy`

### Install Software and Configure (ansible) - `winlab_install.yml`

* Update variables in `all.yml.example` and rename

    ```bash
    mv all.yml.example all.yml
    ```

* Test connectivity to the VMs

    ```bash
    ansible all -i inventory.yml -m win_ping -vvv
    ```

* Run the playbook to configure the servers

    ```bash
    ansible-playbook winlab_install.yml
    ```

> **Note**
> Limit running ansible plays against only connection server and replica hosts and output in verbose `ansible-playbook winlab_install.yml --limit "cs,csr" -vvv`

> **Note**
> The connection servers are in a unconfigured state and at version 8.4.x.

![Horizon Connection Servers1](https://ha.rickrocklin.com/local/winlab/horizon_ui_connection_servers.png)

### Upgrade Connection Servers (ansible) - `connection_server_upgrade.yml`

* Update binary parameter in `all.yml`

    ```bash
    #install_binary: "VMware-Horizon-Connection-Server-x86_64-8.4.1-20741546.exe"
    install_binary: "VMware-Horizon-Connection-Server-x86_64-8.7.0-20649599.exe"
    ```
* Run the playbook to configure the servers

    ```bash
    ansible-playbook connection_server_upgrade.yml
    ```

![Horizon Connection Servers2](https://ha.rickrocklin.com/local/winlab/horizon_ui_connection_servers2.png)

## Credentials

* The local administrator for the VMs:

  | Username | Password |
  | -------- | -------- |
  | `administrator` | `Password1234`|

* The domain accounts:

  | Username | Password |
  | -------- | -------- |
  | `windows_lab\horizonadmin` | `Password1234`|

## Servers

  | Server | IP | Purpose |
  | -------- | -------- | -------- |
  | `dc01.windows_lab.local` | `192.168.20.50` | `Domain Controller` |
  | `cs01.windows_lab.local` | `192.168.20.101` | `Connection Server (standard)` |
  | `cs02.windows_lab.local` | `192.168.20.102` | `Connection Server (replica1)` |
  | `cs03.windows_lab.local` | `192.168.20.103` | `Connection Server (replica2)` |

## References
* Stefan Zimmermann [GitLab](https://gitlab.com/StefanZ8n/packer-ws2022) [Article](https://z8n.eu/2021/11/09/building-a-windows-server-2022-ova-with-packer/)
* Dmitry Teslya [GitHub](https://github.com/dteslya/win-iac-lab)
* Austin Cinderblook [GitHub](https://github.com/Cinderblook/tacklebox)
* [Install Horizon Connection Server Silently - 2209](https://docs.vmware.com/en/VMware-Horizon/2209/horizon-installation/GUID-3790D978-3D71-4D25-8A36-D8F2A9838B7C.html)
  * [Silent Installation Properties for a Horizon Connection Server Standard Installation](https://docs.vmware.com/en/VMware-Horizon/2209/horizon-installation/GUID-56F893BE-91D0-44CF-9C5B-26E28926C3F8.html)
* [Horizon View Connection Server with Ansible](https://www.codecrusaders.nl/devops/ansible/horizon-view-connection-server-with-ansible/)
* [ansible.windows.win_package module ??? Installs/uninstalls an installable package](https://docs.ansible.com/ansible/latest/collections/ansible/windows/win_package_module.html#examples)

## Scratchpad
### Jenkins install

* deploy a GCP VM (free tier)
```bash
gcloud compute instances create gcp-docker-01 --project=jenkins-372604 --zone=us-west1-b --machine-type=e2-micro --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=871997622931-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=jenkins-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-1804-bionic-v20221201,mode=rw,size=10,type=projects/jenkins-372604/zones/us-west1-b/diskTypes/pd-standard --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

* Connect to the VM

```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=$(gcloud config get-value compute/zone)
echo -e "PROJECT ID: $PROJECT_ID\nZONE: $ZONE"

gcloud compute instances list

gcloud compute ssh gcp-docker-01
```

* Install docker: https://github.com/ntalekt/vagrant-ubuntu-docker/blob/master/scripts/docker.sh

### TODO
* Parameterize the installation and upgrade ansible roles. Possible to condense into a single connection server installation role and have the parameters control the installation type?
* install_connection_server_release.sh - continue to complete the install script as parameters exist. MVP only input will be install_connection_server_release.sh -f <pod> -r <test || WLR-nn>
* automation
  * packer image update process
  * connection server vendor download process
  * auto upgrade pod process
  * pod-management pipeline
    * what's the bare minimum that you need to configure in order to provision a single VDI?
    * pod-management pipeline will run every n minutes and try to connect to the pod and configure it, or re-configure it, or do nothing to it, or time out trying to connection to it
* figure out how to git repo for release files would work.
