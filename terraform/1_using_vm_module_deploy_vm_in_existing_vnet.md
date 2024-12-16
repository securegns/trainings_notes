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
