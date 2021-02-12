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