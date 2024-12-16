# Using the Virtual Machine Module
Link - https://learn.acloud.guru/handson/1024e507-c419-4ae2-a9f6-bdcae421b467/course/d40ff22d-673c-48d1-badf-85bdae7892b3

`provider.tf`
```
provider "azurerm" {
  features {}
  skip_provider_registration = true
}
```

`variables.tf`
```
variable "resource_group_name" {
  description = "Name of resource group provided by the lab."
  type        = string
}

variable "vnet_name" {
  description = "Name of the Virtual Network provided by the lab."
  type        = string
}

variable "subnet_name" {
  description = "Name of the subnet to use in the Virtual Network. Defaults to app."
  type        = string
  default     = "app"
}
```

`terraform.tf`
```
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}
```
`main.tf`
```
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

data "azurerm_subnet" "app" {
  resource_group_name  = data.azurerm_resource_group.main.name
  virtual_network_name = var.vnet_name
  name                 = var.subnet_name
}
```
`outputs.tf`
```
output "app_subnet_id" {
  value = data.azurerm_subnet.app.id
}
```

### Deploy an Azure VM using the Virtual Machine Module
`main.tf`
```
resource "tls_private_key" "ssh" {
  algorithm = "RSA"
  rsa_bits  = "4096"
}

resource "azurerm_public_ip" "pip" {

  allocation_method   = "Static"
  location            = data.azurerm_resource_group.main.location
  name                = "pip-${var.vm_name}"
  resource_group_name = data.azurerm_resource_group.main.name
  sku                 = "Standard"
}

module "linux" {
  source  = "Azure/virtual-machine/azurerm"
  version = "1.1.0"

  location                   = data.azurerm_resource_group.main.location
  image_os                   = "linux"
  resource_group_name        = data.azurerm_resource_group.main.name
  allow_extension_operations = false
  boot_diagnostics           = true
  new_network_interface = {
    ip_forwarding_enabled = false
    ip_configurations = [
      {
        public_ip_address_id = azurerm_public_ip.pip.id
        primary              = true
      }
    ]
  }
  admin_username = var.admin_username
  admin_ssh_keys = [
    {
      public_key = tls_private_key.ssh.public_key_openssh
    }
  ]
  name = var.vm_name
  os_disk = {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  os_simple = "UbuntuServer"
  size      = "Standard_F2"
  subnet_id = data.azurerm_subnet.app.id

  custom_data = base64encode(templatefile("${path.module}/custom_data.tpl", {
    admin_username = var.admin_username
    port           = var.application_port
  }))

}
```
`variables.tf`
```
variable "vm_name" {
  description = "Name of virtual machine to create."
  type        = string
}

variable "admin_username" {
  description = "Admin username for virtual machine. Defaults to azureuser."
  type        = string
  default     = "azureuser"
}

variable "application_port" {
  description = "Port to use for the flask application. Defaults to 80."
  type        = number
  default     = 80
}
```
`custom_data.tpl`
```
#!/bin/bash

# Setup logging
logfile="/home/${admin_username}/custom-data.log"
exec > $logfile 2>&1

python3 -V
sudo apt update
sudo apt install -y python3-pip python3-flask
python3 -m flask --version

sudo cat << EOF > /home/${admin_username}/hello.py
from flask import Flask
import requests

app = Flask(__name__)

import requests
@app.route('/')
def hello_world():
    return """<!DOCTYPE html>
<html>
<meta charset="UTF-8">
<head>
    <title>Taco Wagon</title>
</head>
<body>
    <h1>Vroom! Vroom!</h1>
</body>
</html>"""
EOF

chmod +x /home/${admin_username}/hello.py

sudo -b FLASK_APP=/home/${admin_username}/hello.py flask run --host=0.0.0.0 --port=${port}
```

`outputs.tf`
```
output "public_ip" {
  value = azurerm_public_ip.pip.ip_address
}
```
`terraform.tfvars`
```
vm_name             = "taco-wagon-app"
```

### Add a Network Security Group to the VM
`main.tf`
```
resource "azurerm_network_security_group" "app_vm" {
  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location
  name                = "nsg-${var.vm_name}"
}

resource "azurerm_network_security_rule" "http" {
  network_security_group_name = azurerm_network_security_group.app_vm.name
  resource_group_name         = azurerm_network_security_group.app_vm.resource_group_name
  name                        = "http"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  source_address_prefix       = "*"
  destination_port_range      = var.application_port
  destination_address_prefix  = "*"
}

resource "azurerm_network_interface_security_group_association" "app_vm" {
  network_interface_id      = module.linux.network_interface_id
  network_security_group_id = azurerm_network_security_group.app_vm.id
}
```
