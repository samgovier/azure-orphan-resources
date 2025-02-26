# Azure Orphaned Resources - Queries

Here you can find all the orphan resources queries that build this Workbook.

- [Compute](#compute)
  - [App Service Plans](#app-service-plans)
  - [Availability Set](#availability-set)
- [Storage](#storage)
  - [Disks](#disks)
- [Networking](#networking)
  - [Public IPs](#public-ips)
  - [Network Interfaces](#network-interfaces)
  - [Network Security Groups](#network-security-groups)
  - [Route Tables](#route-tables)
  - [Load Balancers](#load-balancers)
  - [Front Door WAF Policy](#front-door-waf-policy)
  - [Traffic Manager Profiles](#traffic-manager-profiles)
  - [Application Gateways](#application-gateways)
  - [Virtual Networks](#virtual-networks)
  - [Subnets](#subnets)
  - [NAT Gateways](#nat-gateways)
  - [IP Groups](#ip-groups)
- [Others](#others)
  - [Resource Groups](#resource-groups)
  - [API Connections](#api-connections)
  - [Certificates](#certificates)

## Compute

#### App Service Plans

App Service plans without hosting Apps.

```kql
resources
| where type =~ "microsoft.web/serverfarms"
| where properties.numberOfSites == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, Sku=sku.name, Tier=sku.tier, tags ,Details
```

#### Availability Set

Availability Sets that not associated to any Virtual Machine (VM) or Virtual Machine Scale Set (VMSS).

```kql
Resources
| where type =~ 'Microsoft.Compute/availabilitySets'
| where properties.virtualMachines == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

## Storage

#### Disks

Managed Disks with 'Unattached' state and not related to Azure Site Recovery.

```kql
Resources
| where type has "microsoft.compute/disks"
| extend diskState = tostring(properties.diskState)
| where managedBy == ""
| where not(name endswith "-ASRReplica" or name startswith "ms-asr-" or name startswith "asrseeddisk-")
| extend Details = pack_all()
| project id, resourceGroup, diskState, sku.name, properties.diskSizeGB, location, tags, subscriptionId, Details
```

> **_Note:_** Azure Site Recovery (aka: ASR) managed disks are excluded from the orphaned resource query.

> <sub> 1) Enable replication process created a temporary *'Unattached'* managed disk that begins with the prefix *"ms-asr-"*.<br/>
        2) When the replication start, a new managed disk that begin with the suffix *"-ASRReplica"* created in *'ActiveSAS'* state.<br/>
        3) When replicated on-premises VMware VMs and physicall servers to managed disks in Azure, these logs are used to create recovery points on Azure-managed disks that have prefix of *"asrseeddisk-"*.</sub>

## Networking

#### Public IPs

Public IPs that are not attached to any resource (VM, NAT Gateway, Load Balancer, Application Gateway, Public IP Prefix, etc.).

```kql
Resources
| where type == "microsoft.network/publicipaddresses"
| where properties.ipConfiguration == "" and properties.natGateway == "" and properties.publicIPPrefix == ""
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, sku.name, tags ,Details
```

#### Network Interfaces

Network Interfaces that are not attached to any resource.

```kql
Resources
| where type has "microsoft.network/networkinterfaces"
| where isnull(properties.privateEndpoint)
| where isnull(properties.privateLinkService)
| where properties.hostedWorkloads == "[]"
| where properties !has 'virtualmachine'
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, tags, subscriptionId, Details
```

> **_Note:_** Azure Netapp Volumes are excluded from the orphaned resource query.

> <sub> When creating a _Volume_ in _Azure Netapp Account_: <br/>
        1) A delegated subnet created in the virtaul network (vNET). <br/>
        2) A Network Interface created in the subnet with the fields: <br/><sub>
&nbsp;&nbsp;&nbsp;&nbsp;- "linkedResourceType": "Microsoft.Netapp/volumes" <br/>
&nbsp;&nbsp;&nbsp;&nbsp;- "hostedWorkloads": ["/subscriptions/<_**SubscriptionId**_>/resourceGroups/<_**RG-Name**_>/providers/Microsoft.NetApp/netAppAccounts/<_**NetAppAccount-Name**_>/capacityPools/<NetAppCapacityPool-Name>/volumes/<_**NetAppVolume-Name**_>" <br/>
&nbsp;&nbsp;&nbsp;&nbsp;- "bareMetalServer": { "id": "/subscriptions/<_**SubscriptionId**_>/resourceGroups/<_**RG-Name**_>/providers/Microsoft.Network/bareMetalServers/<_**baremetalTenant_svm_ID**_>"}</sub></sub>

#### Network Security Groups

Network Security Group (NSGs) that are not attached to any network interface or subnet.

```kql
Resources
| where type == "microsoft.network/networksecuritygroups" and isnull(properties.networkInterfaces) and isnull(properties.subnets)
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Route Tables

Route Tables that not attached to any subnet.

```kql
resources
| where type == "microsoft.network/routetables"
| where isnull(properties.subnets)
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Load Balancers

Load Balancers with empty backend address pools.

```kql
resources
| where type == "microsoft.network/loadbalancers"
| where properties.backendAddressPools == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Front Door WAF Policy

Front Door WAF Policy without associations. (Frontend Endpoint Links, Security Policy Links)

```kql
resources
| where type == "microsoft.network/frontdoorwebapplicationfirewallpolicies"
| where properties.frontendEndpointLinks== "[]" and properties.securityPolicyLinks == "[]"
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, Sku=sku.name, tags, Details
```

#### Traffic Manager Profiles

Traffic Manager without endpoints.

```kql
resources
| where type == "microsoft.network/trafficmanagerprofiles"
| where properties.endpoints == "[]"
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, tags, Details
```

#### Application Gateways

Application Gateways without backend targets. (in backend pools)

```kql
resources
| where type =~ 'microsoft.network/applicationgateways'
| extend backendPoolsCount = array_length(properties.backendAddressPools),SKUName= tostring(properties.sku.name), SKUTier= tostring(properties.sku.tier),SKUCapacity=properties.sku.capacity,backendPools=properties.backendAddressPools , AppGwId = tostring(id)
| project AppGwId, resourceGroup, location, subscriptionId, tags, name, SKUName, SKUTier, SKUCapacity
| join (
    resources
    | where type =~ 'microsoft.network/applicationgateways'
    | mvexpand backendPools = properties.backendAddressPools
    | extend backendIPCount = array_length(backendPools.properties.backendIPConfigurations)
    | extend backendAddressesCount = array_length(backendPools.properties.backendAddresses)
    | extend backendPoolName  = backendPools.properties.backendAddressPools.name
    | extend AppGwId = tostring(id)
    | summarize backendIPCount = sum(backendIPCount) ,backendAddressesCount=sum(backendAddressesCount) by AppGwId
) on AppGwId
| project-away AppGwId1
| where  (backendIPCount == 0 or isempty(backendIPCount)) and (backendAddressesCount==0 or isempty(backendAddressesCount))
| extend Details = pack_all()
| project Resource=AppGwId, resourceGroup, location, subscriptionId, SKUTier, SKUCapacity, tags, Details
```

#### Virtual Networks

Virtual Networks (VNETs) without subnets.

```kql
resources
| where type == "microsoft.network/virtualnetworks"
| where properties.subnets == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Subnets

Subnets without Connected Devices. (Empty Subnets)

```kql
resources
| where type =~ "microsoft.network/virtualnetworks"
| extend subnet = properties.subnets
| mv-expand subnet
| extend Details = pack_all()
| where isnull(subnet.properties.ipConfigurations)
| project subscriptionId, Resource=subnet.id, VNetName=name, SubnetName=tostring(subnet.name) ,resourceGroup, location, Details
```

#### NAT Gateways

NAT Gateways that not attached to any subnet.

```kql
resources
| where type == "microsoft.network/natgateways"
| where isnull(properties.subnets)
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tostring(sku.name), tostring(sku.tier), tags, Details
```

#### IP Groups

IP Groups that not attached to any Azure Firewall.

```kql
resources
| where type == "microsoft.network/ipgroups"
| where properties.firewalls == "[]" and properties.firewallPolicies == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

## Others

#### Resource Groups

Resource Groups without resources (including hidden types resources).

```kql
ResourceContainers
 | where type == "microsoft.resources/subscriptions/resourcegroups"
 | extend rgAndSub = strcat(resourceGroup, "--", subscriptionId)
 | join kind=leftouter (
     Resources
     | extend rgAndSub = strcat(resourceGroup, "--", subscriptionId)
     | summarize count() by rgAndSub
 ) on rgAndSub
 | where isnull(count_)
 | extend Details = pack_all()
 | project subscriptionId, Resource=id, location, tags ,Details
```

#### API Connections

API Connections that not related to any Logic App.

```kql
resources
| where type =~ 'Microsoft.Web/connections'
| project resourceId = id , apiName = name, subscriptionId, resourceGroup, tags, location
| join kind = leftouter (
    resources
    | where type == 'microsoft.logic/workflows'
    | extend resourceGroup, location, subscriptionId, properties
    | extend var_json = properties["parameters"]["$connections"]["value"]
	| mvexpand var_connection = var_json
    | where notnull(var_connection)
    | extend connectionId = extract("connectionId\":\"(.*?)\"", 1, tostring(var_connection))
    | project connectionId, name
    )
    on $left.resourceId == $right.connectionId
| where connectionId == ""
| extend Details = pack_all()
| project resourceId, resourceGroup, location, subscriptionId, tags, Details
```

#### Certificates

Expired certificates.

```kql
resources
| where type == 'microsoft.web/certificates'
| extend expiresOn = todatetime(properties.expirationDate)
| where expiresOn <= now()
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, tags, Details
```
