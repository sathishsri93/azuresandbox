# #AzureSandbox - terraform-azurerm-vnet-opnprem

## Contents

* [Architecture](#architecture)
* [Overview](#overview)
* [Before you start](#before-you-start)
* [Getting started](#getting-started)
* [Smoke testing](#smoke-testing)
* [Documentation](#documentation)

## Architecture

![vnet-onprem-diagram](./vnet-onprem-diagram.drawio.svg)

## Overview

This configuration simulates connectivity to an on-premises network using a site-to-site VPN connection and Azure DNS private resolver. It includes the following resources:

* Simulated on-premises environment
  * A [virtual network](https://learn.microsoft.com/azure/azure-glossary-cloud-terminology#vnet) for hosting [virtual machines](https://learn.microsoft.com/azure/azure-glossary-cloud-terminology#vm).
  * A [VPN gateway site-to-site VPN](https://learn.microsoft.com/en-us/azure/vpn-gateway/design#s2smulti) connection to simulate connectivity from an on-premises network to Azure.
  * A [bastion](https://learn.microsoft.com/azure/bastion/bastion-overview) for secure RDP and SSH access to virtual machines.
  * A Windows Server [virtual machine](https://learn.microsoft.com/azure/azure-glossary-cloud-terminology#vm) running [Active Directory Domain Services](https://learn.microsoft.com/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) with a pre-configured domain and DNS server.
  * A Windows Server [virtual machine](https://learn.microsoft.com/azure/azure-glossary-cloud-terminology#vm) for use as a jumpbox.
* Azure Sandbox environment
  * A [Virtual WAN site-to-site VPN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about#s2s) connection to simulate connectivity from Azure to an on-premises network.
  * A [DNS private resolver](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) is used to resolve DNS queries for private zones in both environments (on-premises and Azure) in a bi-directional fashion.

## Before you start

[terraform-azurerm-vwan](../../terraform-azurerm-vwan/) must be provisioned first before starting.

## Getting started

This section describes how to provision this configuration using default settings.

* Change the working directory.

  ```bash
  cd ~/azuresandbox/extras/terraform-azurerm-vnet-onprem
  ```

* Add an environment variable containing the password for the service principal.

  ```bash
  export TF_VAR_arm_client_secret=YourServicePrincipalSecret
  ```

* Run [bootstrap.sh](./bootstrap.sh) using the default settings or custom settings.

  ```bash
  ./bootstrap.sh
  ```

* Apply the Terraform configuration.

  ```bash
  # Initialize terraform providers
  terraform init

  # Validate configuration files
  terraform validate

  # Review plan output
  terraform plan

  # Apply configuration
  terraform apply
  ```

* Monitor output. Upon completion, you should see a message similar to the following:

  `Apply complete! Resources: 24 added, 0 changed, 0 destroyed.`

* Inspect `terraform.tfstate`.

  ```bash
  # List resources managed by terraform
  terraform state list 
  ```

## Smoke testing

### Test connectivity from cloud to on-premises

* Connect to *jumpwin1* using Bastion.
  * From the client environment, navigate to *portal.azure.com* > *Virtual machines* > *jumpwin1*
  * Click *Connect*.
  * Expand *More ways to connect*.
  * Click *Go to Bastion*
  * For *Username* enter `bootstrapadmin@mysandbox.local`.
  * For *Authentication Type* choose `Password from Azure Key Vault`.
  * For *Subscription* select the appropriate subscription.
  * For *Azure Key Vault* select the appropriate key vault, e.g. `kv-xxxxxxxxxxxxxxx`.
  * For *Azure Key Vault Secret* select `adminpassword`.
  * Click *Connect*.
  * For *password* use the value of the *adminpassword* secret in key vault.
  * Click *Connect*

* Test cloud to on-premises private DNS zones from *jumpwin2*.
  * Navigate to *Start* > *Windows PowerShell*.
  * Run the following PowerShell Command using the FQDN copied previously:

  ```powershell
  Resolve-DnsName jump2win2.myonprem.local
  ```

  * Verify the *IPAddress* returned is within the IP address prefix for *azurerm_subnet.vnet_shared_02_subnets["snet-misc-03"]*, e.g. `192.168.2.4`.
  * Note: This demonstrates that DNS queries can be resolved from the cloud network to the on-premises network. This is accomplished using a DNS forwarding ruleset that is linked to `vnet-app-01` which forwards DNS queries to a private on-premises DNS zone `myonprem.local` to `192.168.1.4` which is the IP address for DNS server `adds2` on the on-premises network.

* Test cloud to on-premises RDP connectivity from *jumpwin1*
  * Navigate to *Start* > *Windows Accessories* > *Remote Desktop Connection*
  * For *Computer*, enter `jumpwin2.myonprem.local`.
  * For *User name*, enter `boostrapadmin@myonprem.local`.
  * Click *Connect*.
  * For *Password*, enter the value of the *adminpassword* secret in key vault.
  * Click *OK*.
  * Close Server Manager.
  * Note: This demonstrates that network traffic can be routed from the cloud network to the on-premises network, and that RDP connections can be established from the cloud to hosts on the on-premises network.

### Test connectivity from on-premises to cloud

* Look up the FQDN for Azure Files endpoint from client environment.
  * Navigate to *portal.azure.com* > *Storage accounts* > *stgxxxxxxxxxxxxxxx* > *Settings* > *Endpoints* > *File service* and copy the FQDN for the `File service`, e.g. `https://stxxxxxxxxxxxxx.file.core.windows.net/`.
  * This will be used in the next smoke test.

* Test on-premises to cloud private DNS zones from *jumpwin2*.
  * Navigate to *Start* > *Windows PowerShell*.
  * Run the following PowerShell Command:

  ```powershell
  # Replace the FQDN below with the FQDN copied previously.
  Resolve-DnsName stxxxxxxxxxxxxx.file.core.windows.net
  ```

  * Verify the *IP4Address* returned is within the IP address prefix for *zurerm_subnet.vnet_app_01_subnets["snet-privatelink-01"]*, e.g. `10.2.2.4`.
  * Note: This demonstrates that DNS queries can be resolved from the on-premises network to the cloud network. This is accomplished using conditional forwarder for `file.core.windows.net` which forwards DNS queries to the inbound endpoint of the DNS private resolver on the cloud network at `10.1.2.4`. The DNS private resolver then queries an Azure private DNS zone to return the Azure Files private endpoint at `10.2.2.4`.

* Test on-premises to cloud SMB connectivity from *jumpwin2*.
  * Navigate to *Start* > *Windows PowerShell*.
  * Run the following PowerShell Command:

  ```powershell
  # Replace the FQDN below with the FQDN copied previously.
  net use z: \\stxxxxxxxxxxx.file.core.windows.net\myfileshare /USER:bootstrapadmin@mysandbox.local
  ```
  
  * For *Password*, enter the value of the *adminpassword* secret in key vault.
  * Create some test files and folders on the newly mapped Z: drive.
  * Unmap the z: drive using the following command:

  ```powershell
  net use z: /delete
  ```

  * Note: This demonstrates that network traffic can be routed from the on-premises network to the cloud network, and that SMB connections can be established from the on-premises network to an Azure Files private endpoint in the cloud network.

## Documentation

This section provides additional information on various aspects of this configuration.

### Bootstrap script

This configuration uses the script [bootstrap.sh](./bootstrap.sh) to create a `terraform.tfvars` file for generating and applying Terraform plans. For simplified deployment, several runtime defaults are initialized using output variables stored in the `terraform.tfstate` files associated with the following configurations:

* [terraform-azurerm-vnet-shared](../../terraform-azurerm-vnet-shared/)
* [terraform-azurerm-vnet-app](../../terraform-azurerm-vnet-app/)
* [terraform-azurerm-vwan](../../terraform-azurerm-vwan/)

Output variable | Configuration | Sample value
--- | --- | ---
aad_tenant_id | terraform-azurerm-vnet-shared | "00000000-0000-0000-0000-000000000000"
adds_domain_name_cloud | terraform-azurerm-vnet-shared | "mysandbox.local"
admin_password_secret | terraform-azurerm-vnet-shared | "adminpassword"
admin_username_secret | terraform-azurerm-vnet-shared | "adminuser"
arm_client_id | terraform-azurerm-vnet-shared | "00000000-0000-0000-0000-000000000000"
automation_account_name | terraform-azurerm-vnet-shared | "auto-xxxxxxxxxxxxxxxx-01"
dns_server_cloud | terraform-azurerm-vnet-shared | "10.1.2.4"
key_vault_id | terraform-azurerm-vnet-shared | "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-sandbox-01/providers/Microsoft.KeyVault/vaults/kv-xxxxxxxxxxxxxxx"
key_vault_name | terraform-azurerm-vnet-shared | "kv-xxxxxxxxxxxxxxx"
location | terraform-azurerm-vnet-shared | "eastus"
resource_group_name | terraform-azurerm-vnet-shared | "rg-sandbox-01"
subscription_id | terraform-azurerm-vnet-shared | "00000000-0000-0000-0000-000000000000"
tags | terraform-azurerm-vnet-shared | "tomap( { "costcenter" = "10177772" "environment" = "dev" "project" = "#AzureSandbox" } )"
vnet_app_01_id | terraform-azurerm-vnet-app | "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-sandbox-01/providers/Microsoft.Network/virtualNetworks/vnet-app-01"
vnet_app_01_name | terraform-azurerm-vnet-app | "vnet-app-01"
vnet_shared_01_id | terraform-azurerm-vnet-shared | "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-sandbox-01/providers/Microsoft.Network/virtualNetworks/vnet-shared-01"
vnet_shared_01_name | terraform-azurerm-vnet-shared | "vnet-shared-01"
vnet_shared_01_subnets | terraform-azurerm-vnet-shared | Contains all the subnet definitions.
vwan_01_hub_01_id | terraform-azurerm-vwan | "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-sandbox-01/providers/Microsoft.Network/virtualHubs/vhub-xxxxxxxxxxxxxxxx-01"
vwan_01_id | terraform-azurerm-vwan |"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-sandbox-01/providers/Microsoft.Network/virtualWans/vwan-xxxxxxxxxxxxxxxx-01"

### Terraform resources

This section describes the resources included in this configuration.

#### Network resources (on-premises)

The configuration for these resources can be found in [020-network-onprem.tf](./020-network-onprem.tf).

Resource name (ARM) | Notes
--- | ---
azurerm_virtual_network.vnet_shared_02 (vnet&#x2011;onprem&#x2011;01) | By default this virtual network is configured with an address space of `192.168.0.0/16` and is configured with DNS server addresses of `192.168.2.4` (the private ip for *azurerm_windows_virtual_machine.vm_adds*) and [168.63.129.16](https://learn.microsoft.com/azure/virtual-network/what-is-ip-address-168-63-129-16). This DNS configuration supports both the default Azure DNS behavior required to initially configure *azurerm_windows_virtual_machine.vm_adds*, as well as the custom DNS behavior provided by the AD integrated DNS server running on *azurerm_windows_virtual_machine.vm_adds* after it is fully configured.
azurerm_subnet.vnet_shared_02_GatewaySubnet | This subnet is reserved for use by *azurerm_virtual_network_gateway.vnet_shared_02_gateway* and has a default IP address prefix of `192.168.0.0/24`. It is defined separately from *azurerm_subnet.vnet_shared_02_subnets* because gateway subnets have special behaviors in Azure such as the restriction on using network security groups.
azurerm_subnet.vnet_shared_02_subnets["AzureBastionSubnet"] | This subnet is reserved for use by *azurerm_bastion_host.bastion_host_02* and has a default IP address prefix of `192.168.1.0/27`. A network security group is associated with this subnet and is configured according to [Working with NSG access and Azure Bastion](https://learn.microsoft.com/azure/bastion/bastion-nsg).
azurerm_subnet.vnet_shared_02_subnets["snet-adds-02"] | This subnet is used by *azurerm_windows_virtual_machine.vm_adds* and has a default IP address prefix of `192.168.2.0/24`. A network security group is associated with this subnet that permits ingress and egress from virtual networks, and egress to the Internet.
azurerm_subnet.vnet_shared_02_subnets["snet-misc-03"] | This subnet is used by *azurerm_windows_virtual_machine.vm_jumpbox_win* and has a default IP address prefix of `192.168.3.0/24`. A network security group is associated with this subnet that permits ingress and egress from virtual networks, and egress to the Internet.
azurerm_bastion_host.bastion_host_02 (bst&#x2011;xxxxxxxxxxxxxxxx&#x2011;2) | Used for secure RDP and SSH access to VMs.
azurerm_virtual_network_gateway.vnet_shared_02_gateway (gw&#x2011;vnet&#x2011;onprem&#x2011;01) | VPN gateway used to connect on-premises network to cloud network.
azurerm_virtual_network_gateway_connection.onprem_to_cloud | Used by *azurerm_virtual_network_gateway.vnet_shared_02_gateway* to connect to cloud network.
azurerm_local_network_gateway.cloud_network | Used by *azurerm_virtual_network_gateway_connection.onprem_to_cloud* to connect to cloud network. *azurerm_vpn_gateway.site_to_site_vpn_gateway_01* is used to configure the gateway address (default `tbd`) and the BGP peering address (default `tbd`).
azurerm_public_ip.vnet_shared_02_gateway_ip (pip&#x2011;vnet&#x2011;onprem&#x2011;01) | Public ip used by *azurerm_virtual_network_gateway.vnet_shared_02_gateway*.
azurerm_public_ip.bastion_host_02 (pip&#x2011;xxxxxxxxxxxxxxxx&#x2011;2) | Public ip used by *azurerm_bastion_host.bastion_host_02*.
random_id.bastion_host_02_name | Used to generate a random name for *azurerm_bastion_host.bastion_host_02*.
random_id.public_ip_bastion_host_02_name | Used to generate a random name for *azurerm_public_ip.bastion_host_02*