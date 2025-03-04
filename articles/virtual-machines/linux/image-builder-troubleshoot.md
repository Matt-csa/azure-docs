---
title: Troubleshoot Azure VM Image Builder
description: This article helps you troubleshoot common problems and errors you might encounter when you're using Azure VM Image Builder.
author: kof-f
ms.author: kofiforson
ms.reviewer: cynthn
ms.date: 10/02/2020
ms.topic: troubleshooting
ms.service: virtual-machines
ms.subservice: image-builder
ms.custom: devx-track-azurepowershell
---

# Troubleshoot Azure VM Image Builder

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Flexible scale sets 

Use this article to troubleshoot and resolve common issues that you might encounter when you're using Azure VM Image Builder.

## Prerequisites

When you're creating a build, do the following:
	
- The VM Image Builder service communicates to the build VM by using WinRM or Secure Shell (SSH). Do *not* disable these settings as part of the build.
- VM Image Builder creates resources as part of the build. Be sure to verify that Azure Policy doesn't prevent VM Image Builder from creating or using necessary resources.
  - Create an IT_ resource group.
  - Create a storage account without a firewall.
- Verify that Azure Policy doesn't install unintended features on the build VM, such as Azure Extensions.
-	Ensure that VM Image Builder has the correct permissions to read/write images and to connect to the storage account. For more information, review the permissions documentation for the [Azure CLI](./image-builder-permissions-cli.md) or [Azure PowerShell](./image-builder-permissions-powershell.md).
- VM Image Builder will fail the build if the scripts or inline commands fail with errors (non-zero exit codes). Ensure that you've tested the custom scripts and verified that they run without error (exit code 0) or require user input. For more information, see [Create an Azure Virtual Desktop image by using VM Image Builder and PowerShell](../windows/image-builder-virtual-desktop.md#tips-for-building-windows-images).

VM Image Builder failures can happen in two areas:
- During image template submission
- During image building

## Troubleshoot image template submission errors

Image template submission errors are returned at submission only. There isn't an error log for image template submission errors. If there's an error during submission, you can return the error by checking the status of the template, specifically by reviewing `ProvisioningState` and `ProvisioningErrorMessage`/`provisioningError`.

```azurecli
az image builder show --name $imageTemplateName  --resource-group $imageResourceGroup
```

```azurepowershell-interactive
Get-AzImageBuilderTemplate -ImageTemplateName  <imageTemplateName> -ResourceGroupName <imageTemplateResourceGroup> | Select-Object ProvisioningState, ProvisioningErrorMessage
```
> [!NOTE]
> For PowerShell, you'll need to install the [VM Image Builder PowerShell modules](../windows/image-builder-powershell.md#prerequisites).

> [!IMPORTANT]
> API version 2021-10-01 introduces a change to the error schema that will be part of every future API release. If you have any Azure VM Image Builder automations, be aware of the [new error output](#error-output-for-version-2021-10-01-and-later) when you switch to API version 2021-10-01 or later. We recommend, after you've switched to the latest API version, that you don't revert to an earlier version, because you'll have to change your automation again to produce the earlier error schema. We don't anticipate that we'll change the error schema again in future releases.

##### **Error output for version 2020-02-14 and earlier**

```
{ 
  "code": "ValidationFailed",
  "message": "Validation failed: 'ImageTemplate.properties.source': Field 'imageId' has a bad value: '/subscriptions/subscriptionID/resourceGroups/resourceGroupName/providers/Microsoft.Compute/images/imageName'. Please review  http://aka.ms/azvmimagebuildertmplref  for details on fields requirements in the Image Builder Template." 
} 
```

##### **Error output for version 2021-10-01 and later**

```
{ 
  "error": {
    "code": "ValidationFailed", 
    "message": "Validation failed: 'ImageTemplate.properties.source': Field 'imageId' has a bad value: '/subscriptions/subscriptionID/resourceGroups/resourceGroupName/providers/Microsoft.Compute/images/imageName'. Please review  http://aka.ms/azvmimagebuildertmplref  for details on fields requirements in the Image Builder Template." 
  }
}
```

The following sections present problem resolution guidance for common image template submission errors.

### Update or upgrade of image templates is currently not supported

#### Error

```text
'Conflict'. Details: Update/Upgrade of image templates is currently not supported
```

#### Cause

The template already exists.

#### Solution

If you submit an image configuration template and the submission fails, a failed template artifact still exists. Delete the failed template.

### The resource operation finished with a terminal provisioning state of "Failed"

#### Error

```text
Microsoft.VirtualMachineImages/imageTemplates 'helloImageTemplateforSIG01' failed with message '{
  "status": "Failed",
  "error": {
    "code": "ResourceDeploymentFailure",
    "message": "The resource operation completed with terminal provisioning state 'Failed'.",
    "details": [
      {
        "code": "InternalOperationError",
        "message": "Internal error occurred."
```
#### Cause

In most cases, the resource deployment failure error occurs because of missing permissions. This error may also be caused by a conflict with the staging resource group.

#### Solution

Depending on your scenario, VM Image Builder might need permissions to:
- The source image or Azure Compute Gallery (formerly Shared Image Gallery) resource group.
- The distribution image or Azure Compute Gallery resource.
- The storage account, container, or blob that the `File` customizer is accessing. 

Also, ensure the staging resource group name is uniquely specified for each image template.

For more information about configuring permissions, see [Configure VM Image Builder permissions by using the Azure CLI](image-builder-permissions-cli.md) or [Configure VM Image Builder permissions by using PowerShell](image-builder-permissions-powershell.md).

### Error getting a managed image

#### Error

```text
Build (Managed Image) step failed: Error getting Managed Image '/subscriptions/.../providers/Microsoft.Compute/images/mymanagedmg1': Error getting managed image (...): compute.
ImagesClient#Get: Failure responding to request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error.
Status=403 Code="AuthorizationFailed" Message="The client '......' with object id '......' doesn't have authorization to perform action 'Microsoft.Compute/images/read' over scope 
```
#### Cause

Missing permissions.

#### Solution

Depending on your scenario, VM Image Builder might need permissions to:
- The source image or Azure Compute Gallery resource group.
- The distribution image or Azure Compute Gallery resource.
- The storage account, container, or blob that the `File` customizer is accessing. 

For more information about configuring permissions, see [Configure VM Image Builder permissions by using the Azure CLI](image-builder-permissions-cli.md) or [Configure VM Image Builder permissions by using PowerShell](image-builder-permissions-powershell.md).

### The build step failed for the image version

#### Error

```text
Build (Shared Image Version) step failed for Image Version '/subscriptions/.../providers/Microsoft.Compute/galleries/.../images/... /versions/0.23768.4001': Error getting Image Version '/subscriptions/.../resourceGroups/<rgName>/providers/Microsoft.Compute/galleries/.../images/.../versions/0.23768.4001': Error getting image version '... :0.23768.4001': compute.GalleryImageVersionsClient#Get: Failure responding to request: StatusCode=404 -- Original Error: autorest/azure: Service returned an error. 
Status=404 Code="ResourceNotFound" Message="The Resource 'Microsoft.Compute/galleries/.../images/.../versions/0.23768.4001' under resource group '<rgName>' was not found."
```

#### Cause

VM Image Builder can't locate the source image.

#### Solution

Ensure that the source image is correct and exists in the location of VM Image Builder.

### Downloading an external file to a local file

#### Error

```text
Downloading external file (<myFile>) to local file (xxxxx.0.customizer.fp) [attempt 1 of 10] failed: Error downloading '<myFile>' to 'xxxxx.0.customizer.fp'..
```

#### Cause

The file name or location is incorrect, or the location isn't reachable.

#### Solution

Ensure that the file is reachable. Verify that the name and location are correct.

### Authorization error creating disk

The Azure Image Builder build fails with an authorization error that looks like the following:

#### Error

```text
Attempting to deploy created Image template in Azure fails with an 'The client '6df325020-fe22-4e39-bd69-10873965ac04' with object id '6df325020-fe22-4e39-bd69-10873965ac04' does not have authorization to perform action 'Microsoft.Compute/disks/write' over scope '/subscriptions/<subscriptionID>/resourceGroups/<resourceGroupName>/providers/Microsoft.Compute/disks/proxyVmDiskWin_<timestamp>' or the scope is invalid. If access was recently granted, please refresh your credentials.' 
```
#### Cause

This error is caused when trying to specify a pre-existing resource group and VNet to the Azure Image Builder service with a Windows source image.  

#### Solution

You will need to assign the contributor role to the resource group for the service principal corresponding to Azure Image Builder's first party app by using the CLI command or portal instructions below.

First, validate that the service principal is associated with Azure Image Builder's first party app by using the following CLI command:
```azurecli-interactive
az ad sp show --id {servicePrincipalName, or objectId}
```

Then, to implement this solution using CLI, use the following command:
```azurecli-interactive
az role assignment create -g {ResourceGroupName} --assignee {AibrpSpOid} --role Contributor 
```

To implement this solution in portal, follow the instructions in this documentation: [Assign Azure roles using the Azure portal - Azure RBAC](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current).

For [Step 1: Identify the needed scope](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current#step-1-identify-the-needed-scope): The needed scope is your resource group. 

For [Step 3: Select the appropriate role](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current#step-3-select-the-appropriate-role): The role is Contributor. 

For [Step 4: Select who needs access](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current#step-4-select-who-needs-access): Select member “Azure Virtual Machine Image Builder” 

Then proceed to [Step 6: Assign role](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current#step-6-assign-role) to assign the role.

## Troubleshoot build failures

For image build failures, get the error from the `lastrunstatus`, and then review the details in the *customization.log* file.


```azurecli
az image builder show --name $imageTemplateName  --resource-group $imageResourceGroup
```

```azurepowershell-interactive
Get-AzImageBuilderTemplate -ImageTemplateName  <imageTemplateName> -ResourceGroupName <imageTemplateResourceGroup> | Select-Object LastRunStatus, LastRunStatusMessage
```

### Customization log

When the image build is running, logs are created and stored in a storage account. VM Image Builder creates the storage account in the temporary resource group when you create an image template artifact.

The storage account name uses the pattern IT_\<ImageResourceGroupName\>_\<TemplateName\>_\<GUID\> (for example, *IT_aibmdi_helloImageTemplateLinux01*).

To view the *customization.log* file in the resource group,  select **Storage Account** > **Blobs** > `packerlogs`, select **directory**, and then select the *customization.log* file.


### Understand the customization log

The log is verbose. It covers the image build, including any issues with the image distribution, such as Azure Compute Gallery replication. These errors are surfaced in the error message of the image template status.

The *customization.log* file includes the following stages:

1. *Deploy the build VM and dependencies by using ARM templates to the IT_ staging resource group* stage. This stage includes multiple POSTs to the VM Image Builder resource provider:

    ```text
    Azure request method="POST" request="https://management.azure.com/subscriptions/<subID>/resourceGroups/IT_aibImageRG200_window2019VnetTemplate01_dec33089-1cc3-cccc-cccc-ccccccc/providers/Microsoft.Storage/storageAccounts
    ..
    PACKER OUT ==> azure-arm: Deploying deployment template ...
    ..
    ```

1. *Status of the deployments* stage. This stage includes the status of each resource deployment:

    ```text
    PACKER ERR 2020/04/30 23:28:50 packer: 2020/04/30 23:28:50 Azure request method="GET" request="https://management.azure.com/subscriptions/<subID>/resourcegroups/IT_aibImageRG200_window2019VnetTemplate01_dec33089-1cc3-4505-ae28-6661e43fac48/providers/Microsoft.Resources/deployments/pkrdp51lc0339jg/operationStatuses/08586133176207523519?[REDACTED]" body=""
    ```

1. *Connect to the build VM* stage.

    In Windows, VM Image Builder connects by using WinRM:

    ```text
    PACKER ERR 2020/04/30 23:30:50 packer: 2020/04/30 23:30:50 Waiting for WinRM, up to timeout: 10m0s
    ..
    PACKER OUT     azure-arm: WinRM connected.
    ```

    In Linux, VM Image Builder connects by using SSH:

    ```text
    PACKER OUT ==> azure-arm: Waiting for SSH to become available...
    PACKER ERR 2019/12/10 17:20:51 packer: 2020/04/10 17:20:51 [INFO] Waiting for SSH, up to timeout: 20m0s
    PACKER OUT ==> azure-arm: Connected to SSH!
    ```

1. *Run customizations* stage. When customizations run, you can identify them by reviewing the *customization.log* file. Search for *(telemetry)*.

    ```text
    (telemetry) Starting provisioner windows-update
    (telemetry) ending windows-update
    (telemetry) Starting provisioner powershell
    (telemetry) ending powershell
    (telemetry) Starting provisioner file
    (telemetry) ending file
    (telemetry) Starting provisioner windows-restart
    (telemetry) ending windows-restart
    
    (telemetry) Finalizing. - This means the build hasfinished
    ```
1. *Deprovision* stage. VM Image Builder adds a hidden customizer. This deprovision step is responsible for preparing the VM for deprovisioning. In Windows, it runs `Sysprep` (by using *c:\DeprovisioningScript.ps1*). In Linux, it runs `waagent-deprovision` (by using /tmp/DeprovisioningScript.sh). 

    For example:
    ```text
    PACKER ERR 2020/03/04 23:05:04 [INFO] (telemetry) Starting provisioner powershell
    PACKER ERR 2020/03/04 23:05:04 packer: 2020/03/04 23:05:04 Found command: if( TEST-PATH c:\DeprovisioningScript.ps1 ){cat c:\DeprovisioningScript.ps1} else {echo "Deprovisioning script [c:\DeprovisioningScript.ps1] could not be found. Image build may fail or the VM created from the Image may not boot. Please make sure the deprovisioning script is not accidentally deleted by a Customizer in the Template."}
    ```

1. *Cleanup* stage. After the build has finished, the VM Image Builder resources are deleted.

    ```text
    PACKER ERR ==> azure-arm: Deleting individual resources ...
    ...
    PACKER ERR 2020/02/04 02:04:23 packer: 2020/02/04 02:04:23 Azure request method="DELETE" request="https://management.azure.com/subscriptions/<subId>/resourceGroups/IT_aibDevOpsImg_t_vvvvvvv_yyyyyy-de5f-4f7c-92f2-xxxxxxxx/providers/Microsoft.Network/networkInterfaces/pkrnijamvpo08eo?[REDACTED]" body=""
    ...
    PACKER ERR ==> azure-arm: The resource group was not created by Packer, not deleting ...
    ```
## Tips for troubleshooting script or inline customization
- Test the code before you supply it to VM Image Builder.
- Ensure that Azure Policy and Firewall allow connectivity to remote resources.
- Output comments to the console by using `Write-Host` or `echo`. Doing so lets you search the *customization.log* file.

## Troubleshoot common build errors

### Packer build command failure

#### Error

```text
  "provisioningState": "Succeeded",
  "lastRunStatus": {
   "startTime": "2020-04-30T23:24:06.756985789Z",
   "endTime": "2020-04-30T23:39:14.268729811Z",
   "runState": "Failed",
   "message": "Failed while waiting for packerizer: Microservice has failed: Failed while processing request: Error when executing packerizer: Packer build command has failed: exit status 1. During the image build, a failure has occurred, please review the build log to identify which build/customization step failed. For more troubleshooting steps go to https://aka.ms/azvmimagebuilderts. Image Build log location: https://xxxxxxxxxx.blob.core.windows.net/packerlogs/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx/customization.log. OperationId: xxxxxx-5a8c-4379-xxxx-8d85493bc791. Use this operationId to search packer logs."
```

#### Cause

Customization failure.

#### Solution

Review the log to locate customizer failures. Search for *(telemetry)*. 

For example:
```text
(telemetry) Starting provisioner windows-update
(telemetry) ending windows-update
(telemetry) Starting provisioner powershell
(telemetry) ending powershell
(telemetry) Starting provisioner file
(telemetry) ending file
(telemetry) Starting provisioner windows-restart
(telemetry) ending windows-restart

(telemetry) Finalizing. - This means the build has finished
```

### Time-out exceeded

#### Error

```text
Deployment failed. Correlation ID: xxxxx-xxxx-xxxx-xxxx-xxxxxxxxx. Failed in building/customizing image: Failed while waiting for packerizer: Timeout waiting for microservice to complete: 'context deadline exceeded'
```

#### Cause

The build exceeded the build time-out. This error is seen in the 'lastrunstatus'.

#### Solution

1. Review the *customization.log* file. Identify the last customizer to run. Search for *(telemetry)*, starting from the bottom of the log. 

1. Check script customizations. The customizations might not be suppressing user interaction for commands, such as `quiet` options. For example, `apt-get install -y` results in the script execution waiting for user interaction.

1. If you're using the `File` customizer to download artifacts greater than 20 MB, see workarounds section.

1. Review errors and dependencies in script that might cause the script to wait.

1. If you expect that the customizations need more time, increase the value of [buildTimeoutInMinutes](image-builder-json.md). The default is 4 hours.

1. If you have resource-intensive actions, such as downloading gigabytes (GB) of files, consider the underlying build VM size. The service uses a Standard_D1_v2 VM. The VM has 1 vCPU and 3.5 GB of memory. If you're downloading 50 GB, you'll likely exhaust the VM resources and cause communication failures between VM Image Builder and the build VM. Retry the build with a larger-memory VM by setting the [VM_size](image-builder-json.md#vmprofile).

### Long file download time

#### Error
```text
[086cf9c4-0457-4e8f-bfd4-908cfe3fe43c] PACKER OUT 
myBigFile.zip 826 B / 826000 B  1.00%
[086cf9c4-0457-4e8f-bfd4-908cfe3fe43c] PACKER OUT 
myBigFile.zip 1652 B / 826000 B  2.00%
[086cf9c4-0457-4e8f-bfd4-908cfe3fe43c] PACKER OUT 
..
hours later...
..
myBigFile.zip 826000 B / 826000 B  100.00%
[086cf9c4-0457-4e8f-bfd4-908cfe3fe43c] PACKER OUT 
```
#### Cause 

`File` customizer is downloading a large file.

#### Solution

`File` customizer is suitable only for small (less than 20 MB) file downloads. For larger file downloads, use a script or inline command. For example, in Linux you can use `wget` or `curl`. In Windows, you can use `Invoke-WebRequest`.

### Error waiting on Azure Compute Gallery

#### Error

```text
Deployment failed. Correlation ID: XXXXXX-XXXX-XXXXXX-XXXX-XXXXXX. Failed in distributing 1 images out of total 1: {[Error 0] [Distribute 0] Error publishing MDI to Azure Compute Gallery:/subscriptions/<subId>/resourceGroups/xxxxxx/providers/Microsoft.Compute/galleries/xxxxx/images/xxxxxx, Location:eastus. Error: Error returned from SIG client while publishing MDI to Azure Compute Gallery for dstImageLocation: eastus, dstSubscription: <subId>, dstResourceGroupName: XXXXXX, dstGalleryName: XXXXXX, dstGalleryImageName: XXXXXX. Error: Error waiting on Azure Compute Gallery future for resource group: XXXXXX, gallery name: XXXXXX, gallery image name: XXXXXX.Error: Future#WaitForCompletion: context has been cancelled: StatusCode=200 -- Original Error: context deadline exceeded}
```

#### Cause

VM Image Builder timed out waiting for the image to be added and replicated to Azure Compute Gallery. If the image is being injected into the gallery, you can assume that the image build was successful. However, the overall process failed because VM Image Builder was waiting on Azure Compute Gallery to complete the replication. Even though the build has failed, the replication continues. You can get the properties of the image version by checking the distribution *runOutput*.

```azurecli
$runOutputName=<distributionRunOutput>
az resource show \
    --ids "/subscriptions/$subscriptionID/resourcegroups/$imageResourceGroup/providers/Microsoft.VirtualMachineImages/imageTemplates/$imageTemplateName/runOutputs/$runOutputName"  \
    --api-version=2020-02-14
```

#### Solution

Increase the value of `buildTimeoutInMinutes`.
 
### Low Windows resource information events

#### Error

```text
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for operation to complete (system performance: 1% cpu; 37% memory)...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for operation to complete (system performance: 51% cpu; 35% memory)...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for operation to complete (system performance: 21% cpu; 37% memory)...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for operation to complete (system performance: 21% cpu; 36% memory)...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for operation to complete (system performance: 90% cpu; 32% memory)...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT     azure-arm: Waiting for the Windows Modules Installer to exit...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer: 2020/04/30 23:38:58 [INFO] command 'PowerShell -ExecutionPolicy Bypass -OutputFormat Text -File C:/Windows/Temp/packer-windows-update-elevated.ps1' exited with code: 101
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT ==> azure-arm: Restarting the machine...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer: 2020/04/30 23:38:58 [INFO] RPC endpoint: Communicator ended with: 101
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] 1672 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] RPC client: Communicator ended with: 101
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] RPC endpoint: Communicator ended with: 101
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT ==> azure-arm: Waiting for machine to become available...
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] 1672 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] RPC client: Communicator ended with: 101
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer: 2020/04/30 23:38:58 [INFO] starting remote command: shutdown.exe -f -r -t 0 -c "packer restart"
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer: 2020/04/30 23:38:58 [INFO] command 'shutdown.exe -f -r -t 0 -c "packer restart"' exited with code: 0
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer: 2020/04/30 23:38:58 [INFO] RPC endpoint: Communicator ended with: 0
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT ==> azure-arm: A system shutdown is in progress.(1115)
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] RPC client: Communicator ended with: 0
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 [INFO] RPC endpoint: Communicator ended with: 0
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] 0 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:58 packer-provisioner-windows-update: 2020/04/30 23:38:58 [INFO] RPC client: Communicator ended with: 0
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer: 2020/04/30 23:38:59 [INFO] starting remote command: shutdown.exe -f -r -t 60 -c "packer restart test"
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer: 2020/04/30 23:38:59 [INFO] command 'shutdown.exe -f -r -t 60 -c "packer restart test"' exited with code: 1115
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer: 2020/04/30 23:38:59 [INFO] RPC endpoint: Communicator ended with: 1115
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 [INFO] 0 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 [INFO] 40 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 [INFO] RPC client: Communicator ended with: 1115
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 [INFO] RPC endpoint: Communicator ended with: 1115
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer-provisioner-windows-update: 2020/04/30 23:38:59 [INFO] 40 bytes written for 'stderr'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer-provisioner-windows-update: 2020/04/30 23:38:59 [INFO] 0 bytes written for 'stdout'
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer-provisioner-windows-update: 2020/04/30 23:38:59 [INFO] RPC client: Communicator ended with: 1115
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER ERR 2020/04/30 23:38:59 packer-provisioner-windows-update: 2020/04/30 23:38:59 Retryable error: Machine not yet available (exit status 1115)
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT Build 'azure-arm' errored: unexpected EOF
[45f485cf-5a8c-4379-9937-8d85493bc791] PACKER OUT 
```
#### Cause

Resource exhaustion. This issue is commonly seen with Windows Update running with the default build VM size D1_V2.

#### Solution

Increase the build VM size.

### The build finished but no artifacts were created

#### Error

```text
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT Build 'azure-arm' errored: Future#WaitForCompletion: context has been cancelled: StatusCode=200 -- Original Error: context deadline exceeded
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR ==> Some builds didn't complete successfully and had errors:
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT 
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR 2020/04/30 22:29:23 machine readable: azure-arm,error []string{"Future#WaitForCompletion: context has been cancelled: StatusCode=200 -- Original Error: context deadline exceeded"}
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT ==> Some builds didn't complete successfully and had errors:
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR ==> Builds finished but no artifacts were created.
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT --> azure-arm: Future#WaitForCompletion: context has been cancelled: StatusCode=200 -- Original Error: context deadline exceeded
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR 2020/04/30 22:29:23 Cancelling builder after context cancellation context canceled
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT 
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR 2020/04/30 22:29:23 [INFO] (telemetry) Finalizing.
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT ==> Builds finished but no artifacts were created.
[a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER ERR 2020/04/30 22:29:24 waiting for all plugin processes to complete...
Done exporting Packer logs to Azure for Packer prefix: [a170b40d-2d77-4ac3-8719-72cdc35cf889] PACKER OUT
```
#### Cause

The build timed out while it was waiting for the required Azure resources to be created.

#### Solution

Rerun the build to try again.

### Resource not found

#### Error

```text
  "provisioningState": "Succeeded",
  "lastRunStatus": {
   "startTime": "2020-05-01T00:13:52.599326198Z",
   "endTime": "2020-05-01T00:15:13.62366898Z",
   "runState": "Failed",
   "message": "network.InterfacesClient#UpdateTags: Failure responding to request: StatusCode=404 -- Original Error: autorest/azure: Service returned an error. Status=404 Code=\"ResourceNotFound\" Message=\"The Resource 'Microsoft.Network/networkInterfaces/aibpls7lz2e.nic.4609d697-be0a-4cb0-86af-49b6fe877fe1' under resource group 'IT_aibImageRG200_window2019VnetTemplate01_9988723b-af56-413a-9006-84130af0e9df' was not found.\""
  },
```
#### Cause

Missing permissions.

#### Solution

Recheck to ensure that VM Image Builder has all the permissions it requires. 

For more information about configuring permissions, see [Configure VM Image Builder permissions by using the Azure CLI](image-builder-permissions-cli.md) or [Configure VM Image Builder permissions by using PowerShell](image-builder-permissions-powershell.md).

### `Sysprep` timing

#### Error

```text
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: Write-Output '>>> Waiting for GA Service (RdAgent) to start ...'
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: while ((Get-Service RdAgent) -and ((Get-Service RdAgent).Status -ne 'Running')) { Start-Sleep -s 5 }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: Write-Output '>>> Waiting for GA Service (WindowsAzureTelemetryService) to start ...'
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: while ((Get-Service WindowsAzureTelemetryService) -and ((Get-Service WindowsAzureTelemetryService).Status -ne 'Running')) { Start-Sleep -s 5 }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: Write-Output '>>> Waiting for GA Service (WindowsAzureGuestAgent) to start ...'
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: while ((Get-Service WindowsAzureGuestAgent) -and ((Get-Service WindowsAzureGuestAgent).Status -ne 'Running')) { Start-Sleep -s 5 }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: Write-Output '>>> Sysprepping VM ...'
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: if( Test-Path $Env:SystemRoot\system32\Sysprep\unattend.xml ) {
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm:   Remove-Item $Env:SystemRoot\system32\Sysprep\unattend.xml -Force
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: & $Env:SystemRoot\System32\Sysprep\Sysprep.exe /oobe /generalize /quiet /quit
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: while($true) {
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm:   $imageState = (Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\State).ImageState
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm:   Write-Output $imageState
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm:   if ($imageState -eq 'IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE') { break }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm:   Start-Sleep -s 5
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: }
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: Write-Output '>>> Sysprep complete ...'
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: >>> Waiting for GA Service (RdAgent) to start ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: >>> Waiting for GA Service (WindowsAzureTelemetryService) to start ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: >>> Waiting for GA Service (WindowsAzureGuestAgent) to start ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: >>> Sysprepping VM ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_COMPLETE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: Get-Service : Cannot find any service with service name 'WindowsAzureGuestAgent'.
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: At C:\DeprovisioningScript.ps1:6 char:9
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: + while ((Get-Service WindowsAzureGuestAgent) -and ((Get-Service Window ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: +         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm:     + CategoryInfo          : ObjectNotFound: (WindowsAzureGuestAgent:String) [Get-Service], ServiceCommandException
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm:     + FullyQualifiedErrorId : NoServiceFoundForGivenName,Microsoft.PowerShell.Commands.GetServiceCommand
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm:
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT     azure-arm: IMAGE_STATE_UNDEPLOYABLE
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER ERR 2020/05/05 22:26:17 Cancelling builder after context cancellation context canceled
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT Cancelling build after receiving terminated
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER ERR 2020/05/05 22:26:17 packer: 2020/05/05 22:26:17 Cancelling provisioning due to context cancellation: context canceled
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: 
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER ERR 2020/05/05 22:26:17 packer: 2020/05/05 22:26:17 Cancelling hook after context cancellation context canceled
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER OUT ==> azure-arm: The resource group was not created by Packer, deleting individual resources ...
[922bdf36-b53c-4e78-9cd8-6b70b9674685] PACKER ERR ==> azure-arm: The resource group was not created by Packer, deleting individual resources ...
```
#### Cause

The cause might be a timing issue because of the D1_V2 VM size. If customizations are limited and are run in less than three seconds, `Sysprep` commands are run by VM Image Builder to deprovision. When VM Image Builder deprovisions, the `Sysprep` command checks for the *WindowsAzureGuestAgent*, which might not be fully installed and might cause the timing issue. 

#### Solution

To avoid the timing issue, you can increase the VM size or you can add a 60-second PowerShell sleep customization.

### The build is canceled after the context cancelation context is canceled

#### Error
```text
PACKER ERR 2020/03/26 22:11:23 Cancelling builder after context cancellation context canceled
PACKER OUT Cancelling build after receiving terminated
PACKER ERR 2020/03/26 22:11:23 packer-builder-azure-arm plugin: Cancelling hook after context cancellation context canceled
..
PACKER ERR 2020/03/26 22:11:23 packer-builder-azure-arm plugin: Cancelling provisioning due to context cancellation: context canceled
PACKER ERR 2020/03/26 22:11:25 packer-builder-azure-arm plugin: [ERROR] Remote command exited without exit status or exit signal.
PACKER ERR 2020/03/26 22:11:25 packer-builder-azure-arm plugin: [INFO] RPC endpoint: Communicator ended with: 2300218
PACKER ERR 2020/03/26 22:11:25 [INFO] 148974 bytes written for 'stdout'
PACKER ERR 2020/03/26 22:11:25 [INFO] 0 bytes written for 'stderr'
PACKER ERR 2020/03/26 22:11:25 [INFO] RPC client: Communicator ended with: 2300218
PACKER ERR 2020/03/26 22:11:25 [INFO] RPC endpoint: Communicator ended with: 2300218
```
#### Cause

VM Image Builder uses port 22 (Linux) or 5986 (Windows) to connect to the build VM. This occurs when the service is disconnected from the build VM during an image build. The reasons for the disconnection can vary, but enabling or configuring a firewall in the script can block the previously mentioned ports.

#### Solution
Review your scripts for firewall changes or enablement, or changes to SSH or WinRM, and ensure that any changes allow for constant connectivity between the service and the build VM on the previously mentioned ports. For more information, see [VM Image Builder networking options](./image-builder-networking.md).

### JWT errors in log early in the build

#### Error
Early in the build process, the build fails and the log indicates a JSON Web Token (JWT) error:

```text
PACKER OUT Error: Failed to prepare build: "azure-arm"
PACKER ERR 
PACKER OUT 
PACKER ERR * client_jwt will expire within 5 minutes, please use a JWT that is valid for at least 5 minutes
PACKER OUT 1 error(s) occurred:
```

#### Cause

The `buildTimeoutInMinutes` value in the template is set to from 1 to 5 minutes.

#### Solution
As described in [Create an VM Image Builder template](./image-builder-json.md), the time-out must be set to 0 to use the default or set to more than 5 minutes to override the default.  Change the time-out in your template to 0 to use the default or to a minimum of 6 minutes.

### Resource deletion errors

#### Error
Intermediate resources are cleaned up toward the end of the build, and the customization log might show several resource deletion errors:

```text
PACKER OUT ==> azure-arm: Error deleting resource. Will retry.
...
PACKER OUT ==> azure-arm: Error: network.PublicIPAddressesClient#Delete: Failure sending request: StatusCode=0 -- Original Error: Code="PublicIPAddressCannotBeDeleted" Message=...
...
PACKER ERR 2022/03/07 18:43:06 packer-plugin-azure plugin: 2022/03/07 18:43:06 Retryable error: network.SecurityGroupsClient#Delete: Failure sending request: StatusCode=0 -- Original Error: Code="InUseNetworkSecurityGroupCannotBeDeleted"...
```

#### Cause
These error log messages are mostly harmless, because resource deletions are retried several times and, ordinarily, they eventually succeed. You can verify this by continuing to follow the deletion logs until you observe a success message. Alternatively, you can inspect the staging resource group to confirm whether the resource has been deleted.

Making these observations is especially important in build failures, where these error messages might lead you to conclude that they're the reason for the failures, even when the actual errors might be elsewhere.

## DevOps tasks 

### Troubleshoot the task
The task fails only if an error occurs during customization. When this happens, the task reports the failure and leaves the staging resource group, with the logs, so that you can identify the issue. 

To locate the log, you need to know the template name. Go to **pipeline** > **failed build**, and then drill down into the VM Image Builder DevOps task. 

You'll see the log and a template name:

```text
start reading task parameters...
found build at:  /home/vsts/work/r1/a/_ImageBuilding/webapp
end reading parameters
getting storage account details for aibstordot1556933914
created archive /home/vsts/work/_temp/temp_web_package_21475337782320203.zip
Source for image:  { type: 'SharedImageVersion',
  imageVersionId: '/subscriptions/<subscriptionID>/resourceGroups/<rgName>/providers/Microsoft.Compute/galleries/<galleryName>/images/<imageDefName>/versions/<imgVersionNumber>' }
template name:  t_1556938436xxx
``` 

1. Go to the Azure portal, search for the template name in the resource group, and then search for the resource group by typing  **IT_***.
1. Select the storage account name > **blobs** > **containers** > **logs**.

### Troubleshoot successful builds

You might occasionally need to investigate successful builds and  review their logs. As mentioned earlier, if the image build is successful, the staging resource group that contains the logs will be deleted as part of the cleanup. To prevent an automatic cleanup, though, you can introduce a `sleep` after the inline command, and then view the logs as the build is paused. To do so, do the following:
 
1. Update the inline command by adding **Write-Host / Echo “Sleep”**. This gives you time to search in the log.
1. Add a `sleep` value of at least 10 minutes by using a [Start-Sleep](/powershell/module/microsoft.powershell.utility/start-sleep) or `Sleep` Linux command.
1. Use this method to identify the log location, and then keep downloading or checking the log until it gets to `sleep`.


### Operation was canceled

#### Error

```text
2020-05-05T18:28:24.9280196Z ##[section]Starting: Azure VM Image Builder Task
2020-05-05T18:28:24.9609966Z ==============================================================================
2020-05-05T18:28:24.9610739Z Task         : Azure VM Image Builder Test
2020-05-05T18:28:24.9611277Z Description  : Build images using Azure Image Builder resource provider.
2020-05-05T18:28:24.9611608Z Version      : 1.0.18
2020-05-05T18:28:24.9612003Z Author       : Microsoft Corporation
2020-05-05T18:28:24.9612718Z Help         : For documentation, and end to end example, please visit: http://aka.ms/azvmimagebuilderdevops
2020-05-05T18:28:24.9613390Z ==============================================================================
2020-05-05T18:28:26.0651512Z start reading task parameters...
2020-05-05T18:28:26.0673377Z found build at:  d:\a\r1\a\_AppsAndImageBuilder\webApp
2020-05-05T18:28:26.0708785Z end reading parameters
2020-05-05T18:28:26.0745447Z getting storage account details for aibstagstor1565047758
2020-05-05T18:28:29.8812270Z created archive d:\a\_temp\temp_web_package_09737279437949953.zip
2020-05-05T18:28:33.1568013Z Source for image:  { type: 'PlatformImage',
2020-05-05T18:28:33.1584131Z   publisher: 'MicrosoftWindowsServer',
2020-05-05T18:28:33.1585965Z   offer: 'WindowsServer',
2020-05-05T18:28:33.1592768Z   sku: '2016-Datacenter',
2020-05-05T18:28:33.1594191Z   version: '14393.3630.2004101604' }
2020-05-05T18:28:33.1595387Z template name:  t_1588703313152
2020-05-05T18:28:33.1597453Z starting put template...
2020-05-05T18:28:52.9278603Z put template:  Succeeded
2020-05-05T18:28:52.9281282Z starting run template...
2020-05-05T19:33:14.3923479Z ##[error]The operation was canceled.
2020-05-05T19:33:14.3939721Z ##[section]Finishing: Azure VM Image Builder Task
```
#### Cause

If the build wasn't canceled by a user, it was canceled by Azure DevOps User Agent. Most likely, the 1-hour time-out has occurred because of Azure DevOps capabilities. If you're using a private project and agent, you get 60 minutes of build time. If the build exceeds the time-out, DevOps cancels the running task.

For more information about Azure DevOps capabilities and limitations, see [Microsoft-hosted agents](/azure/devops/pipelines/agents/hosted#capabilities-and-limitations).
 
#### Solution

You can host your own DevOps agents or look to reduce the time of your build. For example, if you're distributing to Azure Compute Gallery, you can replicate them to one region or replicate them asynchronously. 

### Slow Windows logon

#### Error

This error might occur when you create a Windows 10 image by using VM Image Builder, create a VM from the image, and then use Remote Desktop Protocol (RDP). You wait several minutes at the first logon screen, and then a blue screen displays the following message:

```text
Please wait for the Windows Modules Installer
```

#### Solution

1. In the image build, check to ensure that:

   * There are no outstanding reboots required by adding a Windows Restart customizer as the last customization.
   * All software installation is complete. 
   
1. Add the [/mode:vm](/windows-hardware/manufacture/desktop/sysprep-command-line-options) option to the default `Sysprep` that VM Image Builder uses. For more information, go to the ["Override the commands"](#override-the-commands) section under "VMs created from VM Image Builder images aren't created successfully."  

 
## VMs created from VM Image Builder images aren't created successfully

By default, VM Image Builder runs *deprovision* code at the end of each image customization phase to *generalize* the image. To generalize an image is to set it up to reuse to create multiple VMs. As part of the process, you can pass in VM settings, such as hostname, username, and so on. In Windows, VM Image Builder runs `Sysprep`, and in Linux, VM Image Builder runs `waagent -deprovision`. 

In Windows, VM Image Builder uses a generic `Sysprep` command. However, this command might not be suitable for every successful Windows generalization. With VM Image Builder, you can customize the `Sysprep` command. Note that VM Image Builder is an image automation tool that's responsible for running `Sysprep` command successfully. But you might need different `Sysprep` commands to make your image reusable. In Linux, VM Image Builder uses a generic `waagent -deprovision+user` command. For more information, see [Microsoft Azure Linux Agent documentation](https://github.com/Azure/WALinuxAgent#command-line-options).

If you're migrating an existing customization and you're using various `Sysprep` or `waagent` commands, you can try the VM Image Builder generic commands. If the VM creation fails, use your previous `Sysprep` or `waagent` commands.

Let's suppose you've used VM Image Builder successfully to create a Windows custom image, but you've failed to create a VM successfully from the image. For example, the VM creation fails to finish or it times out. In this event, do either of the following:
* Review the Windows Server Sysprep documentation.
* Raise a support request with the Windows Server Sysprep Customer Services Support team. They can help troubleshoot your issue and advise you on the correct `Sysprep` command.

### Command locations and file names

In Windows:

```
c:\DeprovisioningScript.ps1
```

In Linux:
```bash
/tmp/DeprovisioningScript.sh
```

### The `Sysprep` command: Windows

```PowerShell
Write-Output '>>> Waiting for GA Service (RdAgent) to start ...'
while ((Get-Service RdAgent).Status -ne 'Running') { Start-Sleep -s 5 }
Write-Output '>>> Waiting for GA Service (WindowsAzureTelemetryService) to start ...'
while ((Get-Service WindowsAzureTelemetryService) -and ((Get-Service WindowsAzureTelemetryService).Status -ne 'Running')) { Start-Sleep -s 5 }
Write-Output '>>> Waiting for GA Service (WindowsAzureGuestAgent) to start ...'
while ((Get-Service WindowsAzureGuestAgent).Status -ne 'Running') { Start-Sleep -s 5 }
Write-Output '>>> Sysprepping VM ...'
if( Test-Path $Env:SystemRoot\system32\Sysprep\unattend.xml ) {
  Remove-Item $Env:SystemRoot\system32\Sysprep\unattend.xml -Force
}
& $Env:SystemRoot\System32\Sysprep\Sysprep.exe /oobe /generalize /quiet /quit
while($true) {
  $imageState = (Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\State).ImageState
  Write-Output $imageState
  if ($imageState -eq 'IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE') { break }
  Start-Sleep -s 5
}
Write-Output '>>> Sysprep complete ...'
```

### The `-deprovision` command: Linux

```bash
/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync
```

### Override the commands

To override the commands, use the PowerShell or shell script provisioners to create the command files with the exact file name and put them in the previously listed directories. VM Image Builder reads these commands and writes output to the *customization.log* file.

## Get support
If you've referred to the guidance and are still having problems, you can open a support case. Be sure to select the correct product and support topic. Doing so will ensure that you're connected with the Azure VM Image Builder support team.

Selecting the case product:
```bash
Product Family: Azure
Product: Virtual Machine Running (Window\Linux)
Support Topic: Azure Features
Support Subtopic: Azure Image Builder
```

## Next steps

For more information, see [VM Image Builder overview](../image-builder-overview.md).
