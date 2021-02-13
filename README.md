## To deploy the resource group
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

Example:
```
az deployment group create --resource-group eu-rg01p --template template.json
```

## To deploy the VNets etc. to the resource group
```
az deployment group create --resource-group <resource-group-name> --template-file <path-to-template>
```

Example:
```
az deployment group create --resource-group eu1-rg01p --template-file template.json --parameters params_web-app-db-nogw.json
```

Add `--confirm-with-what-if` to see changes. 

## Note to self on the mode

Default deployment mode is Incremental. Resources not specified in the ARM template aren't removed. You can change this to Complete which *removes* any resources not specified in the ARM template. To do so add `--mode Complete`.

Although Incremental mode doesn't remove **resources** you don't specify, the same is not the case for properties you don't specify of existing resources: 

> When redeploying an existing resource in incremental mode, all properties are reapplied. The properties aren't incrementally added. A common misunderstanding is to think properties that aren't specified in the template are left unchanged. If you don't specify certain properties, Resource Manager interprets the deployment as overwriting those values. Properties that aren't included in the template are reset to the default values. Specify all non-default values for the resource, not just the ones you're updating. The resource definition in the template always contains the final state of the resource. It can't represent a partial update to an existing resource.