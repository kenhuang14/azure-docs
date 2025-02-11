---
title: 'Azure ExpressRoute: Verify Connectivity - Troubleshooting Guide'
description: This page provides instructions on troubleshooting and validating end-to-end connectivity of an ExpressRoute circuit.
services: expressroute
author: duongau

ms.service: expressroute
ms.topic: troubleshooting
ms.date: 01/07/2022
ms.author: duau
ms.custom: seodec18, devx-track-azurepowershell

---
# Verifying ExpressRoute connectivity
This article helps you verify and troubleshoot ExpressRoute connectivity. ExpressRoute extends an on-premises network into the Microsoft cloud over a private connection that is commonly facilitated by a connectivity provider. ExpressRoute connectivity traditionally involves three distinct network zones, as follows:

- 	Customer Network
-  	Provider Network
-  	Microsoft Datacenter

> [!NOTE]
> In the ExpressRoute direct connectivity model (offered at 10/100 Gbps bandwidth), customers can directly connect to Microsoft Enterprise Edge (MSEE) routers' port. Therefore, in the direct connectivity model, there are only customer and Microsoft network zones.
>


The purpose of this document is to help you identify if and where a connectivity issue exists. Thereby, to help seek support from the appropriate team to resolve an issue. If Microsoft support is needed to resolve an issue, open a support ticket with [Microsoft Support][Support].

> [!IMPORTANT]
> This document is intended to help diagnosing and fixing simple issues. It is not intended to be a replacement for Microsoft support. Open a support ticket with [Microsoft Support][Support] if you are unable to solve the problem using the guidance provided.
>
>

## Overview
The following diagram shows the logical connectivity of a customer network to Microsoft network using ExpressRoute.
[![1]][1]

In the preceding diagram, the numbers indicate key network points. These network points are referenced in this article at times by their associated number. Depending on the ExpressRoute connectivity model--Cloud Exchange Co-location, Point-to-Point Ethernet Connection, or Any-to-any (IPVPN)--the network points 3 and 4 may be switches (Layer 2 devices) or routers (Layer 3 devices). In the direct connectivity model, there are no network points 3 and 4; instead CEs (2) are directly connected to MSEEs via dark fiber. The key network points illustrated are as follows:

1.	Customer compute device (for example, a server or PC)
2.	CEs: Customer edge routers 
3.	PEs (CE facing): Provider edge routers/switches that are facing customer edge routers. Referred to as PE-CEs in this document.
4.	PEs (MSEE facing): Provider edge routers/switches that are facing MSEEs. Referred to as PE-MSEEs in this document.
5.	MSEEs: Microsoft Enterprise Edge (MSEE) ExpressRoute routers
6.	Virtual Network (VNet) Gateway
7.	Compute device on the Azure VNet

If the Cloud Exchange Co-location, Point-to-Point Ethernet, or direct connectivity models are used, CEs (2) establish BGP peering with MSEEs (5). 

If the Any-to-any (IPVPN) connectivity model is used, PE-MSEEs (4) establish BGP peering with MSEEs (5). PE-MSEEs propagate the routes received from Microsoft back to the customer network via the IPVPN service provider network.

> [!NOTE]
>For high availability, Microsoft establishes a fully redundant parallel connectivity between MSEEs (5) and PE-MSEEs (4) pairs. A fully redundant parallel network path is also encouraged between customer network and PE-CEs pair. For more information regarding high availability, see the article [Designing for high availability with ExpressRoute][HA]
>
>

The following are the logical steps, in troubleshooting ExpressRoute circuit:

* [Verify circuit provisioning and state](#verify-circuit-provisioning-and-state)
  
* [Validate Peering Configuration](#validate-peering-configuration)
  
* [Validate ARP](#validate-arp)
  
* [Validate BGP and routes on the MSEE](#validate-bgp-and-routes-on-the-msee)
  
* [Confirm the traffic flow](#confirm-the-traffic-flow)

* [Test private peering connectivity](#test-private-peering-connectivity)


## Verify circuit provisioning and state
Provisioning an ExpressRoute circuit establishes a redundant Layer 2 connections between CEs/PE-MSEEs (2)/(4) and MSEEs (5). For more information on how to create, modify, provision, and verify an ExpressRoute circuit, see the article [Create and modify an ExpressRoute circuit][CreateCircuit].

>[!TIP]
>A service key uniquely identifies an ExpressRoute circuit. Should you need assistance from Microsoft or from an ExpressRoute partner to troubleshoot an ExpressRoute issue, provide the service key to readily identify the circuit.
>
>

### Verification via the Azure portal
In the Azure portal, open the ExpressRoute circuit page. In the ![3][3] section of the page, the ExpressRoute essentials are listed as shown in the following screenshot:

![4][4]    

In the ExpressRoute Essentials, *Circuit status* indicates the status of the circuit on the Microsoft side. *Provider status* indicates if the circuit has been *Provisioned/Not provisioned* on the service-provider side. 

For an ExpressRoute circuit to be operational, the *Circuit status* must be *Enabled* and the *Provider status* must be *Provisioned*.

> [!NOTE]
> After configuring an ExpressRoute circuit, if the *Circuit status* is stuck in not enabled status, contact [Microsoft Support][Support]. On the other hand, if the *Provider status* is stuck in not provisioned status, contact your service provider.
>
>

### Verification via PowerShell
To list all the ExpressRoute circuits in a Resource Group, use the following command:

```azurepowershell
Get-AzExpressRouteCircuit -ResourceGroupName "Test-ER-RG"
```

>[!TIP]
>If you are looking for the name of a resource group, you can get it by listing all the resource groups in your subscription, using the command *Get-AzResourceGroup*
>


To select a particular ExpressRoute circuit in a Resource Group, use the following command:

```azurepowershell
Get-AzExpressRouteCircuit -ResourceGroupName "Test-ER-RG" -Name "Test-ER-Ckt"
```

A sample response is:

```output
Name                             : Test-ER-Ckt
ResourceGroupName                : Test-ER-RG
Location                         : westus2
Id                               : /subscriptions/***************************/resourceGroups/Test-ER-RG/providers/***********/expressRouteCircuits/Test-ER-Ckt
Etag                             : W/"################################"
ProvisioningState                : Succeeded
Sku                              : {
                                    "Name": "Standard_UnlimitedData",
                                    "Tier": "Standard",
                                    "Family": "UnlimitedData"
                                   	}
CircuitProvisioningState         : Enabled
ServiceProviderProvisioningState : Provisioned
ServiceProviderNotes             : 
ServiceProviderProperties        : {
                                    "ServiceProviderName": "****",
                                    "PeeringLocation": "******",
                                    "BandwidthInMbps": 100
                                   	}
ServiceKey                       : **************************************
Peerings                         : []
Authorizations                   : []
```

To confirm if an ExpressRoute circuit is operational, pay particular attention to the following fields:

```output
CircuitProvisioningState         : Enabled
ServiceProviderProvisioningState : Provisioned
```

> [!NOTE]
> After configuring an ExpressRoute circuit, if the *Circuit status* is stuck in not enabled status, contact [Microsoft Support][Support]. On the other hand, if the *Provider status* is stuck in not provisioned status, contact your service provider.
>
>

## Validate Peering Configuration
After the service provider has completed the provisioning the ExpressRoute circuit, multiple eBGP based routing configurations can be created over the ExpressRoute circuit between CEs/MSEE-PEs (2)/(4) and MSEEs (5). Each ExpressRoute circuit can have: Azure private peering (traffic to private virtual networks in Azure), and/or Microsoft peering (traffic to public endpoints of PaaS and SaaS). For more information on how to create and modify routing configuration, see the article [Create and modify routing for an ExpressRoute circuit][CreatePeering].

### Verification via the Azure portal

> [!NOTE]
> In IPVPN connectivity model, service providers handle the responsibility of configuring the peerings (layer 3 services). In such a model, after the service provider has configured a peering and if the peering is blank in the portal, try refreshing the circuit configuration using the refresh button on the portal. This operation will pull the current routing configuration from your circuit. 
>

In the Azure portal, status of an ExpressRoute circuit peering can be checked under the ExpressRoute circuit page. In the ![3][3] section of the page, the ExpressRoute peerings would be listed as shown in the following screenshot:

![5][5]

In the preceding example, as noted Azure private peering is provisioned, but Azure public and Microsoft peerings aren't provisioned. A successfully provisioned peering context would also have the primary and secondary point-to-point subnets listed. The /30 subnets are used for the interface IP address of the MSEEs and CEs/PE-MSEEs. For the peerings that are provisioned, the listing also indicates who last modified the configuration. 

> [!NOTE]
> If enabling a peering fails, check if the primary and secondary subnets assigned match the configuration on the linked CE/PE-MSEE. Also check if the correct *VlanId*, *AzureASN*, and *PeerASN* are used on MSEEs and if these values maps to the ones used on the linked CE/PE-MSEE. If MD5 hashing is chosen, the shared key should be same on MSEE and PE-MSEE/CE pair. Previously configured shared key would not be displayed for security reasons. Should you need to change any of these configuration on an MSEE router, refer to [Create and modify routing for an ExpressRoute circuit][CreatePeering].  
>

> [!NOTE]
> On a /30 subnet assigned for interface, Microsoft will pick the second usable IP address of the subnet for the MSEE interface. Therefore, ensure that the first usable IP address of the subnet has been assigned on the peered CE/PE-MSEE.
>


### Verification via PowerShell
To get the Azure private peering configuration details, use the following commands:

```azurepowershell
$ckt = Get-AzExpressRouteCircuit -ResourceGroupName "Test-ER-RG" -Name "Test-ER-Ckt"
Get-AzExpressRouteCircuitPeeringConfig -Name "AzurePrivatePeering" -ExpressRouteCircuit $ckt
```

A sample response, for a successfully configured private peering, is:

```output
Name                       : AzurePrivatePeering
Id                         : /subscriptions/***************************/resourceGroups/Test-ER-RG/providers/***********/expressRouteCircuits/Test-ER-Ckt/peerings/AzurePrivatePeering
Etag                       : W/"################################"
PeeringType                : AzurePrivatePeering
AzureASN                   : 12076
PeerASN                    : 123##
PrimaryPeerAddressPrefix   : 172.16.0.0/30
SecondaryPeerAddressPrefix : 172.16.0.4/30
PrimaryAzurePort           : 
SecondaryAzurePort         : 
SharedKey                  : 
VlanId                     : 200
MicrosoftPeeringConfig     : null
ProvisioningState          : Succeeded
```

 A successfully enabled peering context would have the primary and secondary address prefixes listed. The /30 subnets are used for the interface IP address of the MSEEs and CEs/PE-MSEEs.

To get the Azure public peering configuration details, use the following commands:

```azurepowershell
$ckt = Get-AzExpressRouteCircuit -ResourceGroupName "Test-ER-RG" -Name "Test-ER-Ckt"
Get-AzExpressRouteCircuitPeeringConfig -Name "AzurePublicPeering" -ExpressRouteCircuit $ckt
```

To get the Microsoft peering configuration details, use the following commands:

```azurepowershell
$ckt = Get-AzExpressRouteCircuit -ResourceGroupName "Test-ER-RG" -Name "Test-ER-Ckt"
Get-AzExpressRouteCircuitPeeringConfig -Name "MicrosoftPeering" -ExpressRouteCircuit $ckt
```

If a peering isn't configured, there would be an error message. A sample response, when the stated peering (Azure Public peering in this example) isn't configured within the circuit:

```azurepowershell
Get-AzExpressRouteCircuitPeeringConfig : Sequence contains no matching element
At line:1 char:1
	+ Get-AzExpressRouteCircuitPeeringConfig -Name "AzurePublicPeering ...
	+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    	+ CategoryInfo          : CloseError: (:) [Get-AzExpr...itPeeringConfig], InvalidOperationException
    	+ FullyQualifiedErrorId : Microsoft.Azure.Commands.Network.GetAzureExpressRouteCircuitPeeringConfigCommand
```

> [!NOTE]
> If enabling a peering fails, check if the primary and secondary subnets assigned match the configuration on the linked CE/PE-MSEE. Also check if the correct *VlanId*, *AzureASN*, and *PeerASN* are used on MSEEs and if these values maps to the ones used on the linked CE/PE-MSEE. If MD5 hashing is chosen, the shared key should be same on MSEE and PE-MSEE/CE pair. Previously configured shared key would not be displayed for security reasons. Should you need to change any of these configuration on an MSEE router, refer to [Create and modify routing for an ExpressRoute circuit][CreatePeering].  
>
>

> [!NOTE]
> On a /30 subnet assigned for interface, Microsoft will pick the second usable IP address of the subnet for the MSEE interface. Therefore, ensure that the first usable IP address of the subnet has been assigned on the peered CE/PE-MSEE.
>

## Validate ARP

The ARP table provides a mapping of the IP address and MAC address for a particular peering. The ARP table for an ExpressRoute circuit peering provides the following information for each interface (primary and secondary):
* Mapping of on-premises router interface ip address to the MAC address
* Mapping of ExpressRoute router interface ip address to the MAC address
* Age of the mapping
ARP tables can help validate layer 2 configuration and troubleshooting basic layer 2 connectivity issues.


See [Getting ARP tables in the Resource Manager deployment model][ARP] document, for how to view the ARP table of an ExpressRoute peering, and for how to use the information to troubleshoot layer 2 connectivity issue.


## Validate BGP and routes on the MSEE

To get the routing table from MSEE on the *Primary* path for the *Private* routing context, use the following command:

```azurepowershell
Get-AzExpressRouteCircuitRouteTable -DevicePath Primary -ExpressRouteCircuitName ******* -PeeringType AzurePrivatePeering -ResourceGroupName ****
```

An example response is:

```output
Network : 10.1.0.0/16
NextHop : 10.17.17.141
LocPrf  : 
Weight  : 0
Path    : 65515

Network : 10.1.0.0/16
NextHop : 10.17.17.140*
LocPrf  : 
Weight  : 0
Path    : 65515

Network : 10.2.20.0/25
NextHop : 172.16.0.1
LocPrf  : 
Weight  : 0
Path    : 123##
```

> [!NOTE]
> If the state of a eBGP peering between an MSEE and a CE/PE-MSEE is in Active or Idle, check if the primary and secondary peer subnets assigned match the configuration on the linked CE/PE-MSEE. Also check if the correct *VlanId*, *AzureAsn*, and *PeerAsn* are used on MSEEs and if these values maps to the ones used on the linked PE-MSEE/CE. If MD5 hashing is chosen, the shared key should be same on MSEE and CE/PE-MSEE pair. Should you need to change any of these configuration on an MSEE router, refer to [Create and modify routing for an ExpressRoute circuit][CreatePeering].
>


> [!NOTE]
> If certain destinations are not reachable over a peering, check the route table of the MSEEs for the corresponding peering context. If a matching prefix (could be NATed IP) is present in the routing table, then check if there are firewalls/NSG/ACLs on the path that are blocking the traffic.
>


The following example shows the response of the command for a peering that doesn't exist:

```azurepowershell
Get-AzExpressRouteCircuitRouteTable : The BGP Peering AzurePublicPeering with Service Key ********************* is not found.
StatusCode: 400
```

## Confirm the traffic flow
To get the combined primary and secondary path traffic statistics--bytes in and out--of a peering context, use the following command:

```azurepowershell
Get-AzExpressRouteCircuitStats -ResourceGroupName $RG -ExpressRouteCircuitName $CircuitName -PeeringType 'AzurePrivatePeering'
```

A sample output of the command is:

```output
PrimaryBytesIn PrimaryBytesOut SecondaryBytesIn SecondaryBytesOut
-------------- --------------- ---------------- -----------------
     240780020       239863857        240565035         239628474
```

A sample output of the command for a non-existent peering is:

```azurepowershell
Get-AzExpressRouteCircuitRouteTable : The BGP Peering AzurePublicPeering with Service Key ********************* is not found.
StatusCode: 400
```

## Test private peering connectivity

Test your private peering connectivity by **counting** packets arriving and leaving the Microsoft edge of your ExpressRoute circuit, on the Microsoft Enterprise Edge (MSEE) devices. This diagnostic tool works by applying an Access Control List (ACL) to the MSEE to count the number of packets that hit specific ACL rules. Using this tool will allow you to confirm connectivity by answering the questions such as:

* Are my packets getting to Azure?
* Are they getting back to on-prem?

### Run test
1. To access this diagnostic tool, select **Diagnose and solve problems** from your ExpressRoute circuit in the Azure portal.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/diagnose-problems.png" alt-text="Screenshot of diagnose and solve problem page from ExpressRoute circuit.":::

1. Select the **Connectivity issues** card under **Common problems**.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/connectivity-issues.png" alt-text="Screenshot of connectivity issues option.":::

1. In the dropdown for *Tell us more about the problem you're experiencing*, select **Connectivity to Azure Private, Azure Public, or Dynamics 365 services.**

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/tell-us-more.png" alt-text="Screenshot of drop-down option for problem user is experiencing.":::

1. Scroll down to the **Test your private peering connectivity** section and expand it.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/test-private-peering.png" alt-text="Screenshot of troubleshooting connectivity issues options.":::

1. Execute the [PsPing](/sysinternals/downloads/psping) test from your on-premises IP address to your Azure IP address and keep it running during the connectivity test.

1. Fill out the fields of the form, making sure to enter the same on-premises and Azure IP addresses used in Step 5. Then select **Submit** and then wait for your results to load. Once your results are ready, review the information for interpreting them below.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/form.png" alt-text="Screenshot of debug ACL form.":::

### Interpreting results
Your test results for each MSEE device will look like the example below. You'll have two sets of results for the primary and secondary MSEE devices. Review the number of matches in and out and use the following scenarios to interpret the results:
* **You see packet matches sent and received on both MSEEs:** This indicates healthy traffic inbound to and outbound from the MSEE on your circuit. If loss is occurring either on-premises or in Azure, it's happening downstream from the MSEE.
* **If testing PsPing from on-premises to Azure *(received)* results show matches, but *sent* results show NO matches:** This indicates that traffic is getting inbound to Azure, but isn't returning to on-prem. Check for return-path routing issues (for example, are you advertising the appropriate prefixes to Azure? Is there a UDR overriding prefixes?).
* **If testing PsPing from Azure to on-premises *(sent)* results show NO matches, but *(received)* results show matches:** This indicates that traffic is getting to on-premises, but isn't getting back. You should work with your provider to find out why traffic isn't being routed to Azure via your ExpressRoute circuit.
* **One MSEE shows NO matches, while the other shows good matches:** This indicates that one MSEE isn't receiving or passing any traffic. It could be offline (for example, BGP/ARP down).

#### Example
```
src 10.0.0.0 dst 20.0.0.0 dstport 3389 (received): 120 matches
src 20.0.0.0 srcport 3389 dst 10.0.0.0 (sent): 120 matches
```
This test result has the following properties:

* IP Port: 3389
* On-prem IP Address CIDR: 10.0.0.0
* Azure IP Address CIDR: 20.0.0.0

## Verify virtual network gateway availability

The ExpressRoute virtual network gateway facilitates the management and control plane connectivity to private link services and private IPs deployed to an Azure virtual network. The virtual network gateway infrastructure is managed by Microsoft and sometimes undergoes maintenance. During a maintenance period, performance of the virtual network gateway may be reduced. You can use the *Diagnose and Solve* experience within the ExpressRoute Circuit page to troubleshoot connectivity issues to the virtual network and reactively detect if recent maintenance events reduced the virtual network gateway capacity.

1. To access this diagnostic tool, select **Diagnose and solve problems** from your ExpressRoute circuit in the Azure portal.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/diagnose-problems.png" alt-text="Screenshot of selecting the diagnose and solve problem page from ExpressRoute circuit.":::

1. Select the **Performance Issues** option.

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/performance-issues.png" alt-text="Screenshot of selecting the performance issue option.":::

1. Wait for the diagnostics to run and interpret the results:

    :::image type="content" source="./media/expressroute-troubleshooting-expressroute-overview/gateway-result.png" alt-text="Screenshot of the diagnostic results.":::
   
    Review if your virtual network gateway recently underwent maintenance. If maintenance occurred during a period when you experienced packet loss or latency, it's possible that the reduced capacity of the virtual network gateway contributed to connectivity issues you're experiencing with the target virtual network. Follow the recommended steps and also consider upgrading the [virtual network gateway SKU](expressroute-about-virtual-network-gateways.md#gwsku) to support a higher network throughput and avoid connectivity issues during future maintenance events.

## Next Steps
For more information or help, check out the following links:

- [Microsoft Support][Support]
- [Create and modify an ExpressRoute circuit][CreateCircuit]
- [Create and modify routing for an ExpressRoute circuit][CreatePeering]

<!--Image References-->
[1]: ./media/expressroute-troubleshooting-expressroute-overview/expressroute-logical-diagram.png "Logical Express Route Connectivity"
[2]: ./media/expressroute-troubleshooting-expressroute-overview/portal-all-resources.png "All resources icon"
[3]: ./media/expressroute-troubleshooting-expressroute-overview/portal-overview.png "Overview icon"
[4]: ./media/expressroute-troubleshooting-expressroute-overview/portal-circuit-status.png "ExpressRoute Essentials sample screenshot"
[5]: ./media/expressroute-troubleshooting-expressroute-overview/portal-private-peering.png "ExpressRoute Essentials sample screenshot"

<!--Link References-->
[Support]: https://portal.azure.com/?#blade/Microsoft_Azure_Support/HelpAndSupportBlade
[CreateCircuit]: ./expressroute-howto-circuit-portal-resource-manager.md
[CreatePeering]: ./expressroute-howto-routing-portal-resource-manager.md
[ARP]: ./expressroute-troubleshooting-arp-resource-manager.md
[HA]: ./designing-for-high-availability-with-expressroute.md
[DR-Pvt]: ./designing-for-disaster-recovery-with-expressroute-privatepeering.md