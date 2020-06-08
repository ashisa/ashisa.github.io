---
title: Building data exchange experience on Azure
tagsL: azure azure-data-share unitty aci powershell
---

It has stuck with me since I heard this phrase - *"data is the new oil"*. With a lot of data comes a need to share it, especially when you want to monetize it.

Azure Data Share is a service that allows you to control and streamline the who, how and when of the data sharing with internal as well as external receipents. Read more about it [here](https://docs.microsoft.com/en-us/azure/data-share/overview).

Here's a schematic of the flow between the data providers and data consumers -

![Azure Data Share Workflow](https://docs.microsoft.com/en-us/azure/data-share/media/data-share-flow.png)

In this post, I am going to describe how I recently did a proof-of-concept of a data exchange functionality on Azure using a combination of Azure Functions, Azure Data Share and a little bit of scripting. While you can use Azure Data Service on its own to do this, I needed to automate a lot of things, especially, on the consumer side of things as the partner I was working with - wanted to leave nothing to surprise.

On a very high-level, the key automation was done in two parts -

1. Automate creation of data shares and datasets as well as initiating an invitation using Azure Functions app
2. Automate the activities that the consumers will have to perform to get their hands on the datasets. This was tricky bit since it had to be as *seamless* as possible.

Let's dive deeper!

## Using Azure Functions app to automate provider side of things ##

As I was aiming for a quick turnaround, I used the Azure Data Share PowerShell Module to quickly automate the creation of data share account, shares as well as data sets. I also used the same function to send an invite to the receipents.

The steps to get things up and running are documented below. You need an Azure CLI environment connected to your Azure subcription to execute these commands.

Run the following commands to create an Azure Functions app with PowerShell runtime and deply the code from GitHub -
```
rgName=datashare0305rg
location=eastus2
storageName=datashare0305storage
functionApp=datashare0305func

#create a resource group
rgId=$(az group create -n $rgName -l $location -o tsv --query id)

#create a storage account
az storage account create -n $storageName -g $rgName -l $location --sku Standard_LRS --kind StorageV2

#create the function app
az functionapp create --consumption-plan-location $location --name $functionApp --os-type Windows --resource-group $rgName --runtime powershell --storage-account $storageName --functions-version 2 --deployment-source-url https://github.com/ashisa/datashare-provider.git

#enable managed identity for this function app and add contributor role on the resource group
az functionapp identity assign -g $rgName -n $functionApp --role owner --scope $rgId
```

If all goes well, the above commands will create an Azure Functions app that contains necessary code to create Azure services that will enable you to start using Azure Data Share service to share your datasets with others.

The data consumers can now visit the following URL to get themselves an invitation and get everything provisioned on their subscription as well - 

```
https://datashare0305func.azurewebsites.net/api/initiate?email=ashisa78@live.com
```

Let's dive deeper in the code that makes this happen!

You can take a look at the code in this GitHub repository - [https://github.com/ashisa/datashare-provider](https://github.com/ashisa/datashare-provider)

The strucuture of the repo is as following -
```
datashare-provider
|
├───consumer    --> Contains code that gets executed on the consumer subscription  
|
├───create      --> This function takes care of setting up the data share account, share, permissions etc
|
├───initialize  --> A dummy function to kick off the dependency installation as well as rehydrate the function app
|
└───invite      --> Contains code that initiate the invite and spins up an ACI to run the consumer script
```

The relevant code the *create* function is as following -
```
Write-Host "Creating data share account..."
$dsAccount=(New-AzDataShareAccount -ResourceGroupName $resourceGroup -Name $dsaccountname -Location $location)

Write-Host "Assigning contributor role on the storage account for the data share account..."
$storageAccount=(Get-AzStorageAccount -StorageAccountName $storageName -ResourceGroupName $resourceGroup)
New-AzRoleAssignment -ObjectId $dsAccount.Identity.PrincipalId -RoleDefinitionName "Storage Blob Data Contributor" -Scope $storageAccount.Id

Write-Host "Creating data share..."
New-AzDataShare -ResourceGroupName $resourceGroup -AccountName $dsaccountname -Name $dssharename -Description "From PowerShell" -TermsOfUse "Testing from PowerShell"

Write-Host "Creating container and dataset..."
New-AzStorageContainer -Container dataset1 -Context $storageAccount.Context

$Script:dataset = New-AzDataShareDataSet -ResourcegroupName $resourceGroup -AccountName $dsaccountname -ShareName $dssharename -Name DataSet1 -StorageAccountResourceId $storageAccount.Id -Container dataset1
```

We create the data share account if it doesn't exist already. We then set up the role assignment on the storage account and create a data share which is then followed up by creating a container and a dataset so that you can test it out.

You can uplaod a file to the container as well if you want to experience the data synchronization as well.

The code from the invite function look like the following -
```
Write-Host "This HTTP triggered function executed successfully."
$invitename = $email -replace "@", ""
$invitename = $invitename -replace "\.", ""

$resourceGroup = "datashare0305rg"
$storageName = "datashare0305storage"
$dsaccountname = "datashare0305acct"
$dssharename = "datashare0305"
$location = "EastUS2"
$scriptUri = "https://raw.githubusercontent.com/ashisa/datashare-provider/master/consumer/ds-consumer.ps1"
New-Variable -Scope Script -Name dataset -Value ""

$ErrorActionPreference = "SilentlyContinue";
$dsAccount=(Get-AzDataShareAccount -ResourceGroupName $resourceGroup -Name $dsaccountname)
$Script:dataset = (Get-AzDataShareDataSet -AccountName $dsaccountname -ResourceGroupName $resourceGroup -ShareName $dssharename -Name DataSet1)

Write-Host "Sending invite..."
$invite = New-AzDataShareInvitation -ResourceGroupName $resourceGroup -AccountName $dsaccountname -ShareName $dssharename -Name "$invitename" -TargetEmail "$email"
$inviteID = $invite.InvitationId

Write-Host "Creating ACI instance..."
$container = New-AzContainerGroup -ResourceGroupName $resourceGroup -Name $invitename -DnsNameLabel $invitename -Image docker.io/ashisa/unitty-ds -OsType Linux -IpAddressType Public -Port @(8080) -Cpu 2 -MemoryInGB 2

Write-Host "Creating redirect header"
$url = "http://$($container.Fqdn):$($container.Ports)/?arg=$($scriptUri)&arg=$($inviteID)&arg=$($Script:dataset.Name)&arg=dataset1&arg=$($Script:dataset.DataSetId)"
Write-Host $url
$header = ConvertFrom-StringData -StringData $("Location = $($url)")
```

This function expects an email ID as a parameter and uses that to create an invitation which is an essential step on the provider side. This step is then followed up by creating an ACI instance which uses the [UniTTY-DS](https://github.com/ashisa/unitty/tree/master/unitty-ds) container and passes along the invitation ID and data set ID as well as the path of the script in the consumer folder in the GitHub repository.

We send back a redirect header so that consumer is now redirected to the ACI instance where the following code us executed -

```
Write-Host ""
Write-Host "Initiating Azure Data Share service provisioning..."
Write-Host "Please follow the instructions below to connect to your Azure subscription."
Write-Host "Connecting to Azure subscription..."
(Connect-AzAccount -UseDeviceAuthentication) | Out-Null
Write-Host ""

Write-Host "Initiating provisioning..."
Write-Host "Creating resource group..." -NoNewline
(New-AzResourceGroup -Name $resourceGroup -Location $location) |Out-Null
Write-Host "Done."

Write-Host "Creating data share account..." -NoNewline
($dsAccount=New-AzDataShareAccount -ResourceGroupName $resourceGroup -Name $dsaccountName -Location $location) |Out-Null
Write-Host "Done."

Write-Host "Assigning contributor role on the storage account for the data share account..." -NoNewline
($storageAccount=New-AzStorageAccount -StorageAccountName $storageName -ResourceGroupName $resourceGroup -Location $location -SkuName Standard_LRS)  |Out-Null
New-AzRoleAssignment -ObjectId $dsAccount.Identity.PrincipalId -RoleDefinitionName "Storage Blob Data Contributor" -Scope $storageAccount.Id  |Out-Null
Write-Host "Done."

#Write-Host "Creating container..." -NoNewline
#Start-Job (New-AzStorageContainer -Container $dsContainer -Context $storageAccount.Context *>&1) |Out-Null
#Write-Host "Done."

Write-Host "Accepting the invite..." -NoNewline
(New-AzDataShareSubscription -ResourceGroupName $resourceGroup -AccountName $dsaccountName -Name $dsshareName -SourceShareLocation $location -InvitationId $inviteID)  |Out-Null
Write-Host "Done."

Write-Host "Creating DataSet mapping..." -NoNewline
(New-AzDataShareDataSetMapping -ResourceGroupName $resourceGroup -AccountName $dsaccountName -StorageAccountResourceId $storageAccount.Id -Container $dsContainer -Name $dsshareName -ShareSubscriptionName $dsshareName -DataSetId $datasetId) |Out-Null
Write-Host "Done."

Write-Host "Starting initial snapshot in the background..." -NoNewline
(Start-AzDataShareSubscriptionSynchronization -ResourceGroupName $resourceGroup -AccountName $dsaccountName -ShareSubscriptionName $dsshareName -SynchronizationMode FullSync  -AsJob)  |Out-Null
Write-Host "Done."
```

This was the trickier part as these are the steps are needed to execute on the consumer subscription and the [UniTTY-DS](https://github.com/ashisa/unitty/tree/master/unitty-ds) makes that possible.

This container image downloads the scripts when you navigate to the ACI instance and executes the PowerShell cmdlets in the browser for the consumer. Running a script in a browser is done through the open source project [GoTTY](https://github.com/yudai/gotty).