# What is this?
I had to deploy a 3 tier network to Azure for our various environments (UAT, Prod, etc.) and potentially across multiple geographies later so I created an ARM template to do the same. 

Think of each environment as a vNet for its task and each tier as a subnet in that network. This template isn't limited to 3 tiers though and is flexible. You can specify the names of the tiers you want via the `tierNames` parameter (e.g. `web application database web2 web3`) and it will create a subnet for each. Additionally you can specify if the tier should have a load balancer with public IP or not via the `loadBalancerRequirement` parameter (e.g. `yes yes no no no` - so web & application get load balancers; others do not) and it will create that accordingly. Irrespective of a load balancer being deployed or not each tier also has a (currently empty) NSG created and assigned to the subnet.

Additionally there's the option to have a management tier via the `mgmtSubnetRequired` boolean parameter. This is a subnet without any load balancer or NSG and can be used for management machines. If this is set to `false` the template creates a subnet called `unused` anyways - just in case. :)

The design also called for a "hub" environment - a separate vNet/ environment basically in Azure - that had peerings to each of these environments. This would be where IT had their jumpboxes etc. and where any VPN/ tunnels terminated. Since you need the hub in place before each other environment can create peerings to it, and only after all the environments are present can the hub create peerings to them all there's a parameter called `createPeering` which you can use to control if a peering is attempted when the vNet/ environment is deployed. As I show in the example below initially I run the hub creation with `createPeering` set to `false` and later re-run it with `createPeering` set `true`. 

The ARM template is kind of complicated because I use a bunch of variables and paramters to make it very fluid. I've also gone ahead with IP ranges for all these environments. The template assumes there's six different georgraphies and it sets aside an IP address space for each and within it caters for the environments and subnets. For example in the Europe region we have two sites - North Europe (called `eu1`) and West Europe (called `eu2`). The first one has an address space of 10.110.0.0/16, the second has 10.210.0.0/16. Easy to identify where a VM might be because if it's second octet is in the 100s it is in the primary site (eu1, am1, ap1) while if its in the 200s then its in the secondary site (eu2, am2, ap2). Within this I set divide things such that 10.110.0.0/16 is production, 10.111.0.0/16 is pre-production, and so on ... increments of 1 basically. So given an IP range of 10.213.0.0/16 I know this is in eu2 and in the Dev environment. Finally I carve out /23 subnets with the first one set aside for management/ unused. 

Why keep things simple if I can over-engineer it, eh? :) 

## To deploy the resource group
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Frakheshster%2FnTierARM%2Fmain%2Frgtemplate.json)

**Note**: The only advantage of deploying the resource group via a template is that I can set ACLs and lock it. I'd still have to select a location manually irrespective of whatever friendly location name I choose when deploying or in the paramters file. 

```
az deployment sub create --location <location> --template-file <path-to-template>
```

Use the following to get the ID of the group:
```
az ad group show --group "{name}" --query objectId --output tsv
```

And this for a user ID:
```
az ad user show --id "{email}" --query objectId --output tsv
```

## To deploy the VNets etc. to the resource group
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Frakheshster%2FnTierARM%2Fmain%2Ftemplate.json)

```
az deployment group create --resource-group <resource-group-name> --template-file <path-to-template>
```

Add `--confirm-with-what-if` to see changes. 

## A reminder on the deployment mode

Default deployment mode is Incremental. Resources not specified in the ARM template aren't removed. You can change this to Complete which *removes* any resources not specified in the ARM template. To do so add `--mode Complete`.

Although Incremental mode doesn't remove **resources** you don't specify, the same is not the case for properties you don't specify of existing resources: 

> When redeploying an existing resource in incremental mode, all properties are reapplied. The properties aren't incrementally added. A common misunderstanding is to think properties that aren't specified in the template are left unchanged. If you don't specify certain properties, Resource Manager interprets the deployment as overwriting those values. Properties that aren't included in the template are reset to the default values. Specify all non-default values for the resource, not just the ones you're updating. The resource definition in the template always contains the final state of the resource. It can't represent a partial update to an existing resource.

## Example of how I'd run this:
```
# change this if I am following around and want to create a second or third resourceGroup. 
# note: this does not create multiple resourceGroups, just a single one with that number. 
num="02"

# change these for other sites
site="eu1"
sitecode="northeurope"

# create the resource groups
az deployment sub create --template-file rgtemplate.json --location ${sitecode} --parameters params/${site}-rg${num}h.json
az deployment sub create --template-file rgtemplate.json --location ${sitecode} --parameters params/${site}-rg${num}p.json
az deployment sub create --template-file rgtemplate.json --location ${sitecode} --parameters params/${site}-rg${num}e.json
az deployment sub create --template-file rgtemplate.json --location ${sitecode} --parameters params/${site}-rg${num}u.json
az deployment sub create --template-file rgtemplate.json --location ${sitecode} --parameters params/${site}-rg${num}d.json

# create a vnet and mgmt subnet in the hub resource group. don't do peerings for now. 
az deployment group create --resource-group ${site}-rg${num}h --template-file template.json --parameters params/mgmtonly.json --parameters createPeering=false

# deploy the other vnets for each environment. let's do 3 tiers for each (that's what the param file I point to has)
az deployment group create --resource-group ${site}-rg${num}p --template-file template.json --parameters params/web-app-db-mgmt.json
az deployment group create --resource-group ${site}-rg${num}e --template-file template.json --parameters params/web-app-db-mgmt.json
az deployment group create --resource-group ${site}-rg${num}u --template-file template.json --parameters params/web-app-db-mgmt.json
az deployment group create --resource-group ${site}-rg${num}d --template-file template.json --parameters params/web-app-db-mgmt.json

# run the hub resource group vnet deployment again but this time do peerings. 
az deployment group create --resource-group ${site}-rg${num}h --template-file template.json --parameters params/mgmtonly.json --parameters createPeering=true
```

And to delete it all when I am done:
```
# change as required
num="02"
site="eu1"
sitecode="northeurope"

az group delete -n ${site}-rg${num}p -y
az group delete -n ${site}-rg${num}e -y
az group delete -n ${site}-rg${num}u -y
az group delete -n ${site}-rg${num}d -y
az group delete -n ${site}-rg${num}h -y
```