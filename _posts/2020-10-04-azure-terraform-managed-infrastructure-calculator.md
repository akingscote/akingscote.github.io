---
title: "Azure Terraform Managed Infrastructure Calculator"
date: "2020-10-04"
categories: 
  - "development"
  - "python"
tags: 
  - "azure"
  - "devops"
  - "iac"
  - "python"
  - "sdk"
  - "terraform"
  - "terragrunt"
coverImage: "image.png"
---

I use terraform with Azure for all the obvious benefits that Infrastructure As Code provides. However, sometimes it makes much more sense to provision resources via the frontend portal, or via the AZ cli for speed, unsupported terraform resources or just for experimenting. This got me thinking, how much of my deployed infrastructure is actually managed via terraform?

So i wrote a little proof of concept application using the Azure Python SDK. It has a dependency on managing your terraform state in an Azure cloud storage account (as is recommended by terraform).

Firstly, you'll need to ensure that you set the following environment variables to a service principal which has appropriate permissions to view the resources you intend to query (including Azure Active Directory Groups).
```
export ARM_SUBSCRIPTION_ID="xxxx"
export ARM_CLIENT_ID="xxxxxx"
export ARM_CLIENT_SECRET="xxxxxxxx"
export ARM_TENANT_ID="xxxxxxxx"
```

The subscription id is in reference to the subscription where the storage account that contains the terraform state resides. An alternative method would be to generate a Shared Access Token (SAS) so that the storage account can be accessed regardless of subscription permissions.

You will also need to update config.json to include all the subscription IDs you want to scrape, as well as explicitly state the storage account name. This is just proof of concept code and could easily be improved to be more autonomous.

So once you've setup the variables and configuration file, you should be ready to run. Ensure the python packages are installed:
```
pip3 install azure-storage-blob azure-mgmt-storage>=3.0.0 msrestazure azure-identity
```

And then simply run the iacPercentageCalc.py application.
```
python3 iacPercentage.py
...
INFO - 180 resources in azure
INFO - 118 resources not in terraform
INFO - 34.44% managed by terraform
```

You can also run with a debug flag to improve verbosity to see exactly what resources are being compared.
```
python3 iacPercentage.py -l DEBUG
...
DEBUG - /subscriptions/xxxxxxxxxxxxxxxxxxxxxxx/resourceGroups/xxxxxxxx/providers/Microsoft.Network/virtualNetworks/xxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
DEBUG - xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx is present in terraform state
INFO - 180 resources in azure
INFO - 118 resources not in terraform
INFO - 34.44% managed by terraform
```

## How Does it Work
The application uses the Azure Graph API to generate an OAuth token. Once the token is successfully generated, a HTTP request is fired to the Azure management API to retrieve all resources under the subscriptions specified in the configuration file.

All resources are queried for their \`id\` value which is added to an array. Once the subscriptions have been scraped, the same principal is applied to the Azure Graph application endpoint to retrieve all service principals and then to the groups endpoint to retrieve all Azure Active Directory group ids. So at the end of this process, we have all the id values of all resources (within subscriptions) and all app registrations and AAD groups stored in an array. Now now all ID values are the same, some ID values are UUIDs and others are full URLs to the resource with a UUID embedded within it.

Once the Azure resoruces are retrieved, the application then needs to figure out what resources terraform knows about. The application uses the Azure SDK to connect to the specified storage account (terraform shared state) and retrieves the storage accounts first key (key1). This key is then used in a subsequent request to query the storage account. The storage account is then queried and searches for all output values within the terraform state blob. If the output has an ID value (which they all should) then this value is stored into an array. The terraform state is recursively iterated to capture all ID values.

Now the terraform state does not delete values if they are no longer used. For example, if i created a terraform resource called "test" but then deleted the resource via the portal, the state file would still have a record of that resource, it would just be outdated.

Finally, the application iterates through all the azure resources and takes each ID value and see's if its in the terraform list. If the entry is not in the terraform list, then terraform dosent know anything about it so its not provisioned via terraform. The unmanaged list size is compared against the all azure resource list and a percentage is derived.

Working in the cloud provides new opportunities for information assurance and this is a cool little idea to help provide better insight into your infrastrucure. What you have deployed is not the same as what you think you have deployed, so this application helps to address those gaps.

Full code is available [here](https://gitlab.com/ashleykingscote/azure-iac-calc)
