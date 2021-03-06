---
title: Configure Managed Service Identity on an Azure virtual machine scale set using a template
description: Step-by-step instructions for configuring a Managed Service Identity on an Azure VMSS, using an Azure Resource Manager template.
services: active-directory
documentationcenter: ''
author: daveba
manager: mtillman
editor: ''

ms.service: active-directory
ms.component: msi
ms.devlang: na
ms.topic: conceptual
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 02/20/2018
ms.author: daveba
---

# Configure Managed Service Identity on virtual machine scale using a template

[!INCLUDE[preview-notice](../../../includes/active-directory-msi-preview-notice.md)]

Managed Service Identity provides Azure services with an automatically managed identity in Azure Active Directory. You can use this identity to authenticate to any service that supports Azure AD authentication, without having credentials in your code. 

In this article, you learn how to perform the following Managed Service Identity operations on an Azure virtual machine scale set, using Azure Resource Manager deployment template:
- Enable and disable the system assigned identity on an Azure virtual machine scale set
- Add and remove a user assigned identity on an Azure virtual machine scale set

## Prerequisites

- If you're unfamiliar with Managed Service Identity, check out the [overview section](overview.md). **Be sure to review the [difference between a system assigned and user assigned identity](overview.md#how-does-it-work)**.
- If you don't already have an Azure account, [sign up for a free account](https://azure.microsoft.com/free/) before continuing.
- To perform the management operations in this article, your account needs the following role assignments:
    - [Virtual Machine Contributor](/azure/role-based-access-control/built-in-roles#virtual-machine-contributor) to create a virtual machine scale set and enable and remove system and/or user assigned managed identity from a virtual machine scale set.
    - [Managed Identity Contributor](/azure/role-based-access-control/built-in-roles#managed-identity-contributor) role to create a user assigned identity.
    - [Managed Identity Operator](/azure/role-based-access-control/built-in-roles#managed-identity-operator) role to assign and remove a user assigned identity from and to a virtual machine scale set.

## Azure Resource Manager templates

As with the Azure portal and scripting, [Azure Resource Manager](../../azure-resource-manager/resource-group-overview.md) templates provide the ability to deploy new or modified resources defined by an Azure resource group. Several options are available for template editing and deployment, both local and portal-based, including:

   - Using a [custom template from the Azure Marketplace](../../azure-resource-manager/resource-group-template-deploy-portal.md#deploy-resources-from-custom-template), which allows you to create a template from scratch, or base it on an existing common or [QuickStart template](https://azure.microsoft.com/documentation/templates/).
   - Deriving from an existing resource group, by exporting a template from either [the original deployment](../../azure-resource-manager/resource-manager-export-template.md#view-template-from-deployment-history), or from the [current state of the deployment](../../azure-resource-manager/resource-manager-export-template.md#export-the-template-from-resource-group).
   - Using a local [JSON editor (such as VS Code)](../../azure-resource-manager/resource-manager-create-first-template.md), and then uploading and deploying by using PowerShell or CLI.
   - Using the Visual Studio [Azure Resource Group project](../../azure-resource-manager/vs-azure-tools-resource-groups-deployment-projects-create-deploy.md) to both create and deploy a template.  

Regardless of the option you choose, template syntax is the same during initial deployment and redeployment. Enabling Managed Service Identity on a new or existing VM is done in the same manner. Also, by default, Azure Resource Manager does an [incremental update](../../azure-resource-manager/deployment-modes.md) to deployments.

## System assigned identity

In this section, you will enable and disable the system assigned identity using an Azure Resource Manager template.

### Enable system assigned identity during creation the creation of a virtual machines scale set or a existing virtual machine scale set

1. Whether you sign in to Azure locally or via the Azure portal, use an account that is associated with the Azure subscription that contains the virtual machine scale set.
   
2. To enable the system assigned identity, load the template into an editor, locate the `Microsoft.Compute/virtualMachinesScaleSets` resource of interest within the resources section and add the `identity` property at the same level as the `"type": "Microsoft.Compute/virtualMachines"` property. Use the following syntax:

   ```JSON
   "identity": { 
       "type": "SystemAssigned"
   }
   ```

3. (Optional) Add the virtual machine scale set Managed Service Identity extension as an `extensionsProfile` element. This step is optional as you can use the Azure Instance Metadata Service (IMDS) identity, to retrieve tokens as well.  Use the following syntax:

   >[!NOTE] 
   > The following example assumes a Windows virtual machine scale set extension (`ManagedIdentityExtensionForWindows`) is being deployed. You can also configure for Linux by using `ManagedIdentityExtensionForLinux` instead, for the `"name"` and `"type"` elements.
   >

   ```json
   "extensionProfile": {
        "extensions": [
            {
                "name": "MSIWindowsExtension",
                "properties": {
                    "publisher": "Microsoft.ManagedIdentity",
                    "type": "ManagedIdentityExtensionForWindows",
                    "typeHandlerVersion": "1.0",
                    "autoUpgradeMinorVersion": true,
                    "settings": {
                        "port": 50342
                    },
                    "protectedSettings": {}
                }
            }
   ```

4. When you're done, the following sections should added to the resource section of your template  and should resemble the following:

   ```json
    "resources": [
        {
            //other resource provider properties...
            "apiVersion": "2018-06-01",
            "type": "Microsoft.Compute/virtualMachineScaleSets",
            "name": "[variables('vmssName')]",
            "location": "[resourceGroup().location]",
            "identity": {
                "type": "SystemAssigned",
            },
           "properties": {
                //other resource provider properties...
                "virtualMachineProfile": {
                    //other virtual machine profile properties...
                    "extensionProfile": {
                        "extensions": [
                            {
                                "name": "ManagedIdentityWindowsExtension",
                                "properties": {
                                  "publisher": "Microsoft.ManagedIdentity",
                                  "type": "ManagedIdentityExtensionForWindows",
                                  "typeHandlerVersion": "1.0",
                                  "autoUpgradeMinorVersion": true,
                                  "settings": {
                                      "port": 50342
                                  }
                                }
                            } 
                        ]
                    }
                }
            }
        }
    ]
   ``` 

### Disable a system assigned identity from an Azure virtual machine scale set

If you have a virtual machine scale set that no longer needs a managed service identity:

1. Whether you sign in to Azure locally or via the Azure portal, use an account that is associated with the Azure subscription that contains the virtual machine scale set.

2. Load the template into an [editor](#azure-resource-manager-templates) and locate the `Microsoft.Compute/virtualMachineScaleSets` resource of interest within the `resources` section. If you have a VM that only has system assigned identity, you can disable it by changing the identity type to `None`.

   **Microsoft.Compute/virtualMachineScaleSets API version 2018-06-01**

   If your apiVersion is `2018-06-01` and your VM has both system and user assigned identities, remove `SystemAssigned` from the identity type and keep `UserAssigned` along with the userAssignedIdentities dictionary values.

   **Microsoft.Compute/virtualMachineScaleSets API version 2018-06-01 and earlier**

   If your apiVersion is `2017-12-01` and your virtual machine scale set has both system and user assigned identities, remove `SystemAssigned` from the identity type and keep `UserAssigned` along with the `identityIds` array of the user assigned identities. 
   
    

   The following example shows you how remove a system assigned identity from a virtual machine scale set with no user assigned identities:
   
   ```json
   {
       "name": "[variables('vmssName')]",
       "apiVersion": "2018-06-01",
       "location": "[parameters(Location')]",
       "identity": {
           "type": "None"
        }

   }
   ```

## User assigned identity

In this section, you assign a user assigned identity to a virtual machine scale set using Azure Resource Manager template.

> [!Note]
> To create a user assigned identity using an Azure Resource Manager Template, see [Create a user assigned identity](how-to-manage-ua-identity-arm.md#create-a-user-assigned-identity).

### Assign a user assigned identity to an Azure VMSS

1. Under the `resources` element, add the following entry to assign a user assigned identity to your virtual machine scale set.  Be sure to replace `<USERASSIGNEDIDENTITY>` with the name of the user assigned identity you created.
   
   **Microsoft.Compute/virtualMachineScaleSets API version 2018-06-01**

   If your apiVersion is `2018-06-01`, your user assigned identities are stored in the `userAssignedIdentities` dictionary format and the `<USERASSIGNEDIDENTITYNAME>` value must be stored in a variable defined in the `variables` section of your template.

   ```json
   {
       "name": "[variables('vmssName')]",
       "apiVersion": "2018-06-01",
       "location": "[parameters(Location')]",
       "identity": {
           "type": "userAssigned",
           "userAssignedIdentities": {
               "[resourceID('Microsoft.ManagedIdentity/userAssignedIdentities/',variables('<USERASSIGNEDIDENTITYNAME>'))]": {}
           }
       }
    
   }
   ```   

   **Microsoft.Compute/virtualMachineScaleSets API version 2017-12-01**
    
   If your `apiVersion` is `2017-12-01` or earlier, your user assigned identities are stored in the `identityIds` array and the `<USERASSIGNEDIDENTITYNAME>` value must be stored in a variable defined in the variables section of your template.

   ```json
   {
       "name": "[variables('vmssName')]",
       "apiVersion": "2017-03-30",
       "location": "[parameters(Location')]",
       "identity": {
           "type": "userAssigned",
           "identityIds": [
               "[resourceID('Micrososft.ManagedIdentity/userAssignedIdentities/',variables('<USERASSIGNEDIDENTITY>'))]"
           ]
       }

   }
   ``` 

2. (Optional) Add the following entry under the `extensionProfile` element to assign the managed identity extension to your VMSS. This step is optional as you can use the Azure Instance Metadata Service (IMDS) identity endpoint, to retrieve tokens as well. Use the following syntax:
   
    ```JSON
       "extensionProfile": {
            "extensions": [
                {
                    "name": "MSIWindowsExtension",
                    "properties": {
                        "publisher": "Microsoft.ManagedIdentity",
                        "type": "ManagedIdentityExtensionForWindows",
                        "typeHandlerVersion": "1.0",
                        "autoUpgradeMinorVersion": true,
                        "settings": {
                            "port": 50342
                        },
                        "protectedSettings": {}
                    }
                }
    ```

3. When you are done, your template should look similar to the following:
   
   **Microsoft.Compute/virtualMachineScaleSets API version 2018-06-01**   

   ```json
   "resources": [
        {
            //other resource provider properties...
            "apiVersion": "2018-06-01",
            "type": "Microsoft.Compute/virtualMachineScaleSets",
            "name": "[variables('vmssName')]",
            "location": "[resourceGroup().location]",
            "identity": {
                "type": "UserAssigned",
                "userAssignedIdentities": {
                    "[resourceID('Microsoft.ManagedIdentity/userAssignedIdentities/',variables('<USERASSIGNEDIDENTITYNAME>'))]": {}
                }
            },
           "properties": {
                //other virtual machine properties...
                "virtualMachineProfile": {
                    //other virtual machine profile properties...
                    "extensionProfile": {
                        "extensions": [
                            {
                                "name": "ManagedIdentityWindowsExtension",
                                "properties": {
                                  "publisher": "Microsoft.ManagedIdentity",
                                  "type": "ManagedIdentityExtensionForWindows",
                                  "typeHandlerVersion": "1.0",
                                  "autoUpgradeMinorVersion": true,
                                  "settings": {
                                      "port": 50342
                                  }
                                }
                            } 
                        ]
                    }
                }
            }
        }
    ]
   ```

   **Microsoft.Compute/virtualMachines API version 2017-12-01 eand earlier**

   ```json
   "resources": [
        {
            //other resource provider properties...
            "apiVersion": "2017-12-01",
            "type": "Microsoft.Compute/virtualMachineScaleSets",
            "name": "[variables('vmssName')]",
            "location": "[resourceGroup().location]",
            "identity": {
                "type": "UserAssigned",
                "identityIds": [
                    "[resourceID('Microsoft.ManagedIdentity/userAssignedIdentities/',variables('<USERASSIGNEDIDENTITYNAME>'))]"
                ]
            },
           "properties": {
                //other virtual machine properties...
                "virtualMachineProfile": {
                    //other virtual machine profile properties...
                    "extensionProfile": {
                        "extensions": [
                            {
                                "name": "ManagedIdentityWindowsExtension",
                                "properties": {
                                  "publisher": "Microsoft.ManagedIdentity",
                                  "type": "ManagedIdentityExtensionForWindows",
                                  "typeHandlerVersion": "1.0",
                                  "autoUpgradeMinorVersion": true,
                                  "settings": {
                                      "port": 50342
                                  }
                                }
                            } 
                        ]
                    }
                }
            }
        }
    ]
   ```
### Remove user assigned identity from an Azure virtual machine scale set

If you have a virtual machine scale set that no longer needs a managed service identity:

1. Whether you sign in to Azure locally or via the Azure portal, use an account that is associated with the Azure subscription that contains the virtual machine scale set.

2. Load the template into an [editor](#azure-resource-manager-templates) and locate the `Microsoft.Compute/virtualMachineScaleSets` resource of interest within the `resources` section. If you have a virtual machine scale set that only has user assigned identity, you can disable it by changing the identity type to `None`.

   The following example shows you how remove all user assigned identities from a VM with no system assigned identities:

   ```json
   {
       "name": "[variables('vmssName')]",
       "apiVersion": "2018-06-01",
       "location": "[parameters(Location')]",
       "identity": {
           "type": "None"
        }
   }
   ```
   
   **Microsoft.Compute/virtualMachineScaleSets API version 2018-06-01**
    
   To remove a single user assigned identity from a virtual machine scale set, remove it from the `userAssignedIdentities` dictionary.

   If you have a system assigned identity, keep it in the in the `type` value under the `identity` value.

   **Microsoft.Compute/virtualMachineScaleSets API version 2017-12-01**

   To remove a single user assigned identity from a virtual machine scale set, remove it from the `identityIds` array.

   If you have a system assigned identity, keep it in the in the `type` value under the `identity` value.
   
## Next steps

- For a broader perspective about Managed Service Identity, read the [Managed Service Identity overview](overview.md).

