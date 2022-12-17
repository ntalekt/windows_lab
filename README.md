# Lab for VMware Horizon

* Deployed an ubuntu desktop via Vagrant in order to keep all the tooling central: [vagrant-ubuntu-desktop](https://github.com/ntalekt/vagrant-ubuntu-desktop)

## Image Creation (packer)

* Update variables in `myvarfile.json.example` and rename

    ```bash
    mv myvarfile.json.example myvarfile.json
    ```

* Update variables in `WinServ2022.pkr.hcl`

* Initialize packer

    ```bash
    packer init -upgrade WinServ2022.pkr.hcl
    ```

* Create the image

    ```bash
    packer build -timestamp-ui -force -var-file=myvarfile.json WinServ2022.pkr.hcl
    ```

## Deploy VMs (terraform)

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

* Actually deploy the VMs (parallelism=1 because my lab is slow)

    ```bash
    terraform apply -auto-approve -parallelism=1
    ```

## Configure VMs (ansible)

* Update variables in `all.yml.example` and rename

    ```bash
    mv all.yml.example all.yml
    ```

* Test connectivity to the VMs

    ```bash
    ansible all -i inventory.yml -m win_ping -vvv
    ```

* 
