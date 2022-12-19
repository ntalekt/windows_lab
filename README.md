# Lab for VMware Horizon

## **Overview**

The goal of this environment is for some VMware Horizon testing. Currently I have a vSphere environment with several Ubuntu servers but zero Windows.

### **Prerequisites**

All the prerequisites are baked into this vagrant ubuntu desktop: [vagrant-ubuntu-desktop](https://github.com/ntalekt/vagrant-ubuntu-desktop)

> **Note**
> If you already have a linux machine and you'd like to install the prerequisites you can look in the following scripts in [vagrant-ubuntu-desktop](https://github.com/ntalekt/vagrant-ubuntu-desktop)
> * [common](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/apt_update.sh)
> * [ansible](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/ansible.sh)
> * [packer & terraform](https://github.com/ntalekt/vagrant-ubuntu-desktop/blob/master/scripts/ansible.sh)

### **Packer**

Packer creates a Windows Server 2022 vSphere template and ovf that has VMware tools, and some other basic applications installed. 

Packer uses `autounattend.xml` and `sysprep-autounattend.xml` to automate Windows Settings windows_lab/packer/configs.

  * It pulls Windows Server 2022 Datacenter Eval Edition (Desktop Experience) from Microsoft's site
  * Installs & configure OpenSSH Client & Server for remote connection
  * Installs VMware tools from ISO provided from the build ESX server

#### **Files**

  * `myvarfile.json` All those quality values that will be used
  * `WinServ2022.pkr.hcl` The main top quality with variable declares at the top, and the provisioner steps after.
  * `scripts/win-update.ps1` runs first: updates windows
  * `scripts/adjustments.ps1` runs second: tweaks windows, and installs some tools
  * `scripts/cleanup.ps1` cleans up windows after all the updates
  * `install-vmware-tools-from-iso.ps1` installs VMware tools so it's baked into the base image.

**Packer Provisioner Steps**
* Updating OS via Windows Update
* Doing some OS adjustments
  * Set Windows telemetry settings to minimum
  * Show file extentions by default
  * Install [Chocolatey](https://chocolatey.org/) - a Windows package manager
    * Install Microsoft Edge (Chromium)
    * Install Win32-OpenSSH-Server
    * Install PowerShell Core
    * Install 7-Zip
    * Install Notepad++
  * Enable Powershell-Core (`pwsh`) to be the default SSHD shell
* Cleanup tasks
* Remove CDROM drives from VM template (otherwise there would be 2)

### **Terraform**

Terraform deploys the required virtual machines against infrastructure using the vSphere provider in this case.

#### **Files**

* `variables.tf` declares the variables that will be used
* `terraform.tfvars` All those quality values that will be used
* `base.tf` defining the vSphere provider and common stuffs
* `01-PDF.tf` defining the Primary Domain Controller VM
* `02-ConnServ.tf` defining the Primary Connection Server VM
* `03-ConnServ2.tf` defining the Second Connection Server VM

### **Ansible**

Configures the VMs once they are deployed. 

* Setup Windows Server Feature: **Domain**
  * Primary Domain Controller
  * Auto-Join the Virutal Machines to the respective Domain created
  * Create a few users and groups within Active Directory
* Install Horizon Connection: Primary
* Install Horizon Connection: Replica
* Common Configurations
  * Enable RDP and allow it through the firewall on all windows servers created

#### **Files**

* `inventory.yml` inventory of the hosts we will be touching
* `winlab_install.yml` an association of the roles to the servers
* `ansible.cfg` main hotness
* `group_vars/all.yml` defines all they key:value pairs needed
* `roles/*` all the different roles and stuff that tells ansible what it needs to do (**hint**: look in `winlab_install.yml`)

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
> This will result in a template in your vSphere infrastructure named WinServ2022 and an ovf in the build directory.

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

* Actually deploy the VMs (parallelism=1 because my lab is slow :dizzy_face:) 

    ```bash
    terraform apply -auto-approve -parallelism=1
    ```

> **Note**
> This will result in 3 VMs being creating using the variables defined in `terraform.tfvars`

```bash
...
vsphere_virtual_machine._PDC: Creation complete after 10m11s [id=420b41aa-e3fc-8ae7-19a2-537ba43fb62b]
...
vsphere_virtual_machine._ConnServ: Creation complete after 10m59s [id=420bf9b2-4ed7-a291-fe8f-df3a07d019ab]
...
vsphere_virtual_machine._ConnServ2: Creation complete after 9m50s [id=420baf8a-8b33-95d0-8720-b0efb2e56f1f]
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```
> **Warning**
> Remove the VMs by running `terraform apply -auto-approve -destroy`

### Configure VMs (ansible)

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
> Limit running ansible plays against only cs and csr hosts and output in verbose `ansible-playbook winlab_install.yml --limit "cs,csr" -vvv`

## References
* Stefan Zimmermann [GitLab](https://gitlab.com/StefanZ8n/packer-ws2022) [Article](https://z8n.eu/2021/11/09/building-a-windows-server-2022-ova-with-packer/)
* Dmitry Teslya [GitHub](https://github.com/dteslya/win-iac-lab)
* Austin Cinderblook [GitHub](https://github.com/Cinderblook/tacklebox)
* [Install Horizon Connection Server Silently - 2209](https://docs.vmware.com/en/VMware-Horizon/2209/horizon-installation/GUID-3790D978-3D71-4D25-8A36-D8F2A9838B7C.html)
  * [Silent Installation Properties for a Horizon Connection Server Standard Installation](https://docs.vmware.com/en/VMware-Horizon/2209/horizon-installation/GUID-56F893BE-91D0-44CF-9C5B-26E28926C3F8.html)
* [Horizon View Connection Server with Ansible](https://www.codecrusaders.nl/devops/ansible/horizon-view-connection-server-with-ansible/)
* [ansible.windows.win_package module – Installs/uninstalls an installable package](https://docs.ansible.com/ansible/latest/collections/ansible/windows/win_package_module.html#examples)