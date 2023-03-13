---
title: Authorize access to files using Active Directory
titleSuffix: Azure Storage
description: Authorize access to Azure files using Azure Active Directory (Azure AD). Assign Azure roles for access rights. Access data with an Azure AD account.
author: 

ms.service: storage
ms.topic: conceptual
ms.date: 03/09/2023
ms.author: 
ms.subservice: files
---

# Overview of Azure Files identity-based authentication using Azure Active Directory for REST access (Preview)

This article explains how Azure file shares can use Azure Active Directory based authentication and authorization to support identity-based access to Azure file shares over REST API interface. 

## Glossary
It's helpful to understand some key terms relating to identity-based authentication for Azure file shares:

- **Azure Active Directory (Azure AD)**

    Azure AD is Microsoft's multi-tenant cloud-based directory and identity management service. Azure AD combines core directory services, application access management, and identity protection into a single solution.
- **Azure role-based access control (Azure RBAC)**

    Azure RBAC enables fine-grained access management for Azure. Using Azure RBAC, you can manage access to resources by granting users the fewest permissions needed to perform their jobs. For more information, see [What is Azure role-based access control]()
 - **Azure REST API**
    
    Representational State Transfer (REST) APIs are service endpoints that support sets of HTTP operations (methods), which provide create/retrieve/update/delete access to the service's resources. See the [Azure REST API Reference]() for more details
 - **Authentication**
    
    The act of granting an authenticated security principal permission to do something. There are two primary use cases in the Azure AD programming model:
    - During an OAuth 2.0 authorization grant flow: when the resource owner grants authorization to the client application, allowing the client to access the resource owner's resources.
    - During resource access by the client: as implemented by the resource server, using the claim values present in the access token to make access control decisions based upon them.

 - **Authorization code**
    
    One of the endpoints implemented by the authorization server, used to interact with the resource owner to provide an authorization grant during an OAuth 2.0 authorization grant flow. Depending on the authorization grant flow used, the actual grant provided can vary, including an authorization code or security token.
See the OAuth 2.0 specification's authorization grant types and authorization endpoint sections, and the OpenIDConnect specification for more details.

 - **Access Token**
    
    A type of security token issued by an authorization server and used by a client application to access a protected resource server. Typically in the form of a JSON Web Token (JWT), the token embodies the authorization granted to the client by the resource owner, for a requested level of access. The token contains all applicable claims about the subject, enabling the client application to use it as a form of credential when accessing a given resource. This also eliminates the need for the resource owner to expose credentials to the client. See the access tokens reference for more details

## Preview Restrictions
- The feature is only supported on data plane APIs that operate on files and directories. It is not supported on data plane APIs that operate on File shares and Storage accounts. It is recommended to use SRM/ARM APIs for managing file share and storage account operations. See the <supported APIs link> for more information.
- Current Azure Files OAuth over REST feature will allow privileged read and write access to data in the file shares. See Privileged access to files data for more details.
- To use the feature, users will have to explicitly pass the intent while requesting for the OAuth token. If the REST API is invoked without intent, the access request will be denied.
- The only supported value for intent is ‘backup’

## Privileged access to files data 
With Azure Files OAuth over REST feature, we have introduced two new data actions named
- Microsoft.Storage/storageAccounts/fileServices/readFileBackupSemantics/action
- Microsoft.Storage/storageAccounts/fileServices/writeFileBackupSemantics/action

Users, groups or service principals requiring OAuth access to Azure Files over REST API interface will have to have either readFileBackupSemantics or writeFileBackupSemantics or both as part of the role definition
When a user, group or service principal calls the REST API with ‘backup’ intent, the system will check if the user or group is assigned one of the new data actions mentioned above. If the user or the group doesn’t have any new data actions in the role, the access will be denied. If the user or the group has the new data actions assigned, the system will bypass any granular file and directory level checks and authorize access to file share data based on RBAC access rules at the share (or higher) scope.

This functionality will provide storage account-wide privileges that will supersede all permissions on files and folders under all file shares in the storage account. It is important to note that any wildcard use cases defined for the following path(s) - Microsoft.Storage/storageAccounts/fileServices/* or higher scope - will automatically inherit the additional access and permissions granted through this new data action. 

To prevent unintended or overprivileged access to Azure Files, we have implemented additional checks that require users/applications to explicitly indicate their intent to use the additional privilege. Furthermore, we strongly recommend that customers review their user RBAC role assignments and replace any wildcard usage with explicit permissions to ensure proper data access management. For more information, please refer to our documentation.

We have also created two new built-in roles that can be used with this feature. 

**1. Storage File Data Privileged Reader** : Has full read access on all the shares for the configured storage account for all the REST API calls using the intent. The system will bypass any file/directory level read denies and allow read access to the data

Actions
[]

Data Actions
[
    
Microsoft.Storage/storageAccounts/fileServices/fileShares/files/read
Microsoft.Storage/storageAccounts/fileServices/readFileBackupSemantics/action

]

**2. Storage File Data Privileged Contributor**: Has full read, write, modify permission and delete access on all the shares for the configured storage account for all the API calls using the intent. The system will bypass any file/directory level read/write/modify permission/delete denies and allow read/write/modify permission/delete access to the data.

Actions
[]

Data Actions
[
    
Microsoft.Storage/storageAccounts/fileServices/fileShares/files/read
Microsoft.Storage/storageAccounts/fileServices/fileShares/files/write
Microsoft.Storage/storageAccounts/fileServices/fileShares/files/delete
Microsoft.Storage/storageAccounts/fileServices/writeFileBackupSemantics/action
Microsoft.Storage/storageAccounts/fileServices/fileshares/files/modifypermissions/action

]

    
## Customer use cases
OAuth authentication and authorization with Azure Files over the REST API interface can benefit customers in various scenarios, including:

### Managed Service Identities 
Customers with applications and managed identities that require access to file share data for backup, restore, or auditing purposes can benefit from OAuth authentication and authorization. Enforcing file and directory level permissions for each identity adds complexity and may not be compatible with certain workloads. 
For instance, customers may want to authorize a backup solution service to access Azure File shares with read-only access to all files with no regard to file-specific permissions.

### Storage Account key replacement
Customers can replace Storage Account Key access with OAuth authentication and authorization to access Azure File shares with read-all/write-all privileges. This approach offers better audit/tracking around access mechanisms and higher security key management.

### Application development and service integrations
OAuth authentication and authorization enable developers to build applications that access Azure storage REST API’s using user or application identities from Azure Active Directory. 
Customers/Partner can also enable 1P and 3P services to securely and transparently configure necessary access to a customer storage account. 
DevOps tools such as the Azure portal, PowerShell and CLI, AZCopy, and storage explorer can manage data using the user’s identity, eliminating the need to handle or distribute all-powerful storage access keys.


## Assign Azure roles for access rights

Azure Active Directory (Azure AD) authorizes access rights to secured resources through Azure RBAC. Azure Storage defines a set of  built-in RBAC roles that encompass common sets of permissions used to access file data. You can also define custom roles for access to file data.

An Azure AD security principal may be a user, a group, an application service principal, or a [managed identity for Azure resources](../../active-directory/managed-identities-azure-resources/overview.md). The RBAC roles that are assigned to a security principal determine the permissions that the principal has for the specified resource. To learn more about assigning Azure roles for file access, see [Assign an Azure role for access to file data](../blobs/assign-azure-role-data-access.md)

In some cases you may need to enable fine-grained access to file resources or to simplify permissions when you have a large number of role assignments for a storage resource. You can use Azure attribute-based access control (Azure ABAC) to configure conditions on role assignments. You can use conditions with a [custom role](../../role-based-access-control/custom-roles.md) or select built-in roles. For more information about configuring conditions for Azure storage resources with ABAC, see [Authorize access to files using Azure role assignment conditions (preview)](../blobs/storage-auth-abac.md). For details about supported conditions for file data operations, see [Actions and attributes for Azure role assignment conditions in Azure Storage (preview)](../blobs/storage-auth-abac-attributes.md).

> [!NOTE]
> When you create an Azure Storage account, you are not automatically assigned permissions to access data via Azure AD. You must explicitly assign yourself an Azure role for access to file Storage. You can assign it at the level of your subscription, resource group, storage account, or file share.

### Resource scope

Before you assign an Azure RBAC role to a security principal, determine the scope of access that the security principal should have. Best practices dictate that it's always best to grant only the narrowest possible scope. Azure RBAC roles defined at a broader scope are inherited by the resources beneath them.

You can scope access to Azure file resources at the following levels, beginning with the narrowest scope:

- **An individual file share.** At this scope, a role assignment applies to a single file share, and to the file share's properties and metadata.
- **The storage account.** At this scope, a role assignment applies to all of the file shares in the storage account, and to the file share' properties and metadata.
- **The resource group.** At this scope, a role assignment applies to all of the file shares in all of the storage accounts in the resource group.
- **The subscription.** At this scope, a role assignment applies to all of the file shares in all of the storage accounts in all of the resource groups in the subscription.
- **A management group.** At this scope, a role assignment applies to all of the file shares in all of the storage accounts in all of the resource groups in all of the subscriptions in the management group.

For more information about scope for Azure RBAC role assignments, see [Understand scope for Azure RBAC](../../role-based-access-control/scope-overview.md).

### Azure built-in roles for files

Azure RBAC provides several built-in roles for authorizing access to file data using Azure AD and OAuth. Some examples of roles that provide permissions to data resources in Azure Storage include:

- [Storage File Data SMB Share Contributor](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner): Allows for read, write, and delete access on files/directories in Azure file shares. This role has no built-in equivalent on Windows file servers.
- [Storage File Data SMB Share Elevated Contributor](../../role-based-access-control/built-in-roles.md#storage-blob-data-contributor): Allows for read, write, delete, and modify ACLs on files/directories in Azure file shares. This role is equivalent to a file share ACL of change on Windows file servers.
- [Storage File Data SMB Share Reader](../../role-based-access-control/built-in-roles.md#storage-blob-data-reader): Allows for read access on files/directories in Azure file shares. This role is equivalent to a file share ACL of read on Windows file servers.
- [Reader and Data Access](../../role-based-access-control/built-in-roles.md#storage-blob-delegator): Lets you view everything but will not let you delete or create a storage account or contained resource. It will also allow read/write access to all data contained in a storage account via access to storage account keys.

To learn how to assign an Azure built-in role to a security principal, see [Assign an Azure role for access to file data](../files/assign-azure-role-data-access.md). To learn how to list Azure RBAC roles and their permissions, see [List Azure role definitions](../../role-based-access-control/role-definitions-list.md).

For more information about how built-in roles are defined for Azure Storage, see [Understand role definitions](../../role-based-access-control/role-definitions.md#control-and-data-actions). For information about creating Azure custom roles, see [Azure custom roles](../../role-based-access-control/custom-roles.md).

Only roles explicitly defined for data access permit a security principal to access file data. Built-in roles such as **Owner**, **Contributor**, and **Storage Account Contributor** permit a security principal to manage a storage account, but don't provide access to the file data within that account via Azure AD. However, if a role includes **Microsoft.Storage/storageAccounts/listKeys/action**, then a user to whom that role is assigned can access data in the storage account via Shared Key authorization with the account access keys. For more information, see [Choose how to authorize access to file data in the Azure portal](../../storage/files/authorize-data-operations-portal.md).

For detailed information about Azure built-in roles for Azure Storage for both the data services and the management service, see the **Storage** section in [Azure built-in roles for Azure RBAC](../../role-based-access-control/built-in-roles.md#storage). Additionally, for information about the different types of roles that provide permissions in Azure, see [Azure roles, Azure AD roles, and classic subscription administrator roles](../../role-based-access-control/rbac-and-directory-admin-roles.md).

> [!IMPORTANT]
> Azure role assignments may take up to 30 minutes to propagate.

### Access permissions for data operations

For details on the permissions required to call specific File service operations, see [Permissions for calling data operations](/rest/api/storageservices/authorize-with-azure-active-directory#permissions-for-calling-data-operations).
    
## Access file data with an Azure AD account

### Use Azure AD to authorize access in application code

The Azure Identity client library simplifies the process of getting an OAuth 2.0 access token for authorization with Azure Active Directory (Azure AD) via the [Azure SDK](https://github.com/Azure/azure-sdk). The latest versions of the Azure Storage client libraries for .NET, Java, Python, JavaScript, and Go integrate with the Azure Identity libraries for each of those languages to provide a simple and secure means to acquire an access token for authorization of Azure Storage requests.

An advantage of the Azure Identity client library is that it enables you to use the same code to acquire the access token whether your application is running in the development environment or in Azure. The Azure Identity client library returns an access token for a security principal. When your code is running in Azure, the security principal may be a managed identity for Azure resources, a service principal, or a user or group. In the development environment, the client library provides an access token for either a user or a service principal for testing purposes.

The access token returned by the Azure Identity client library is encapsulated in a token credential. You can then use the token credential to get a service client object to use in performing authorized operations against Azure Storage. A simple way to get the access token and token credential is to use the **DefaultAzureCredential** class that is provided by the Azure Identity client library. **DefaultAzureCredential** attempts to get the token credential by sequentially trying several different credential types. **DefaultAzureCredential** works in both the development environment and in Azure.

[!INCLUDE [storage-auth-language-table](../../../includes/storage-auth-language-table.md)]

Authorizing file data operations with Azure AD is supported only for REST API versions 2022-11-02 and later. For more information, see [Versioning for the Azure Storage services](/rest/api/storageservices/versioning-for-the-azure-storage-services#specifying-service-versions-in-requests).

#### Sample Code (.NET)
    
    using Azure.Core;
    using Azure.Identity;
    using Azure.Storage.Files.Shares;
    using Azure.Storage.Files.Shares.Models;
 
    namespace FilesOAuthSample
    {
        internal class Program
        {
            static async Task Main(string[] args)
            {
                string tenantId = "";
                string appId = "";
                string appSecret = "";
                string aadEndpoint = "";
                string accountUri = "";
 
                string connectionString = "";
 
                string shareName = "testShare";
                string directoryName = "testDirectory";
                string fileName = "testFile";
 
                ShareClient sharedKeyShareClient = new ShareClient(connectionString, shareName);
                await sharedKeyShareClient.CreateIfNotExistsAsync();
 
                TokenCredential tokenCredential = new ClientSecretCredential(
                    tenantId,
                    appId,
                    appSecret,
                    new TokenCredentialOptions()
                    {
                        AuthorityHost = new Uri(aadEndpoint)
                    });
 
                ShareClientOptions clientOptions = new ShareClientOptions(ShareClientOptions.ServiceVersion.V2021_12_02);
 
                // Set Allow Trailing Dot and Source Allow Trailing Dot.
                clientOptions.AllowTrailingDot = true;
                clientOptions.SourceAllowTrailingDot = true;
 
                // x-ms-file-intent=backup will automatically be applied to all APIs
                // where it is required in derived clients.
                ShareServiceClient shareServiceClient = new ShareServiceClient(
                    new Uri(accountUri),
                    tokenCredential,
                    **ShareFileRequestIntent.Backup**,
                    clientOptions);
 
                ShareClient shareClient = shareServiceClient.GetShareClient(shareName);
 
                ShareDirectoryClient directoryClient = shareClient.GetDirectoryClient(directoryName);
                await directoryClient.CreateAsync();
 
                ShareFileClient fileClient = directoryClient.GetFileClient(fileName);
                await fileClient.CreateAsync(maxSize: 1024);
 
                await fileClient.GetPropertiesAsync();
 
                await sharedKeyShareClient.DeleteIfExistsAsync();
            
            }
        }
    }


### Data access from the Azure portal

The Azure portal can use either your Azure AD account or the account access keys to access file data in an Azure storage account. Which authorization scheme the Azure portal uses depends on the Azure roles that are assigned to you.

[!INCLUDE [storage-shared-key-caution](../../../includes/storage-shared-key-caution.md)]
    
When you attempt to access file data, the Azure portal first checks whether you've been assigned an Azure role with **Microsoft.Storage/storageAccounts/listkeys/action**. If you've been assigned a role with this action, then the Azure portal uses the account key for accessing file data via Shared Key authorization. If you haven't been assigned a role with this action, then the Azure portal attempts to access data using your Azure AD account.

To access file data from the Azure portal using your Azure AD account, you need permissions to access file data, and you also need permissions to navigate through the storage account resources in the Azure portal. The built-in roles provided by Azure Storage grant access to file resources, but they don't grant permissions to storage account resources. For this reason, access to the portal also requires the assignment of an Azure Resource Manager role such as the [Reader](../../role-based-access-control/built-in-roles.md#reader) role, scoped to the level of the storage account or higher. The **Reader** role grants the most restricted permissions, but another Azure Resource Manager role that grants access to storage account management resources is also acceptable. To learn more about how to assign permissions to users for data access in the Azure portal with an Azure AD account, see [Assign an Azure role for access to file data](../files/assign-azure-role-data-access.md).

The Azure portal indicates which authorization scheme is in use when you navigate to a container. For more information about data access in the portal, see [Choose how to authorize access to file data in the Azure portal](../files/authorize-data-operations-portal.md).

### Data access from PowerShell
    
Azure Storage provides extensions for PowerShell that enable you to sign in and run scripting commands with Azure Active Directory (Azure AD) credentials. When you sign in to PowerShell with Azure AD credentials, an OAuth 2.0 access token is returned. That token is automatically used by PowerShell to authorize subsequent data operations against Blob storage. For supported operations, you no longer need to pass an account key or SAS token with the command.

You can assign permissions to blob data to an Azure AD security principal via Azure role-based access control (Azure RBAC). For more information about Azure roles in Azure Storage, see Assign an Azure role for access to file data.

#### Supported operations

The Azure Storage extensions are supported for operations on file data. Which operations you may call depends on the permissions granted to the Azure AD security principal with which you sign in to PowerShell. Permissions to Azure Storage containers are assigned via Azure RBAC. For example, if you have been assigned the Storage File Data SMB Share Reader role, then you can run scripting commands that read data from a file share. If you have been assigned the Storage File Data Privileged Contributor role, then you can run scripting commands that read, write, or delete a file share or the data they contain.

For details on the permissions required to call specific File service operations, see Permissions for calling data operations.

[!IMPORTANT] When a storage account is locked with an Azure Resource Manager ReadOnly lock, the List Keys operation is not permitted for that storage account. List Keys is a POST operation, and all POST operations are prevented when a ReadOnly lock is configured for the account. For this reason, when the account is locked with a ReadOnly lock, users users who do not already possess the account keys must use Azure AD credentials to access blob data. In PowerShell, include the -UseConnectedAccount parameter to create an AzureStorageContext object with your Azure AD credentials.

#### Call PowerShell commands using Azure AD credentials

To use Azure PowerShell to sign in and run subsequent operations against Azure Storage using Azure AD credentials, create a storage context with OAuth to reference the storage account, and include the -UseConnectedAccount parameter.

The storage context with OAuth will only work if it is called with ‘-ShareFileRequestIntent’ option with a value of ‘backup’. This is to specify the explicit intent to use the additional permissions that this feature provides. 

The storage context with OAuth will only work for operations on files and directories. For operations on storage account and file shares, you will have to use the context with storage account key or SAS

#### Sample Code
The following Azure PowerShell example shows how to
- Create a new storage account (using storage account key authorization)
- Create a file share (using storage account authorization)
- Create a test file and test directory in the file share (OAuth authorization using Azure AD credentials)

Remember to replace placeholder values in angle brackets with your own values:

    1.Sign in to your Azure account with the Connect-AzAccount command:
        Connect-AzAccount
        For more information about signing into Azure with PowerShell, see Sign in with Azure PowerShell.

    2.Create an Azure resource group by calling New-AzResourceGroup.
        $resourceGroup = "sample-resource-group-ps"
        $location = "eastus"
        New-AzResourceGroup -Name $resourceGroup -Location $location

    3.Create a storage account by calling New-AzStorageAccount.
        $storageAccount = New-AzStorageAccount -ResourceGroupName $resourceGroup `
        -Name "<storage-account>" `
        -SkuName Standard_LRS `
        -Location $location `

    4.Before you create the file share, assign the appropriate role that has explicit permissions to perform data operations 
    against the storage account. For more  information about assigning Azure roles, see Assign an Azure role for access to file data.

        [!IMPORTANT] Azure role assignments may take a few minutes to propagate.

    5.Get the storage account context using the storage account key by calling Get-AzStorageAccount cmdlet
        $ctxkey = (Get-AzStorageAccount -ResourceGroupName $resouceGroupName -Name $accountname).Context

    6.Create a file share by calling New-AzStorageShare. Because we are using the context created in the previous steps, 
    the file share is created using your storage account key.
        fileshareName = "sample-share"
        New-AzStorageShare -Name $filesahreName -Context $ctxkey

    7.Get the storage account context using OAuth for performing data operations on the file share. 
        $ctx = New-AzStorageContext -StorageAccountName $accountname -ShareFileRequestIntent Backup
        To get the storage account context with OAuth, you need to explicitly pass the -ShareFileRequestIntent Backup parameter 
    to the New-AzStorageContext cmdlet. If the intent parameter is not passed, subsequent file share operation requests using the context will fail
    
    8.Before you create the file or the directory, assign the appropriate role that has explicit permissions to perform data operations 
    against the file share. For more information about assigning Azure roles, see Assign an Azure role for access to file data. 
    To get privileged accessed to file share data make sure the role has the required permissions. For details on the permissions required 
    to call specific File service operations, see Permissions for calling data operations.

    9.Create a test directory and file in the file share using New-AzStorageDirectory and Set-AzStorageFileContent cmdlets respectively. 
    Because this is called using the context created in step 7, the file and directory will be created using Azure AD credentials.
        $dir = New-AzStorageDirectory -ShareName $shareName -Path "dir1" -Context $ctx
        $file = Set-AzStorageFileContent -ShareName $shareName -Path "test2" -Source “<local source file path>” -Context $ctx 

    
### Azure CLI

Azure CLI credentials. After you sign in, your session runs under those credentials. To learn more, see one of the following articles:

<Placeholder for CLI access details. CLI is still under development> [Updated 3/13/2023]

## Feature support

[!INCLUDE [File Storage feature support in Azure Storage accounts](../../../includes/azure-storage-feature-support.md)]

## Next steps

- [Authorize access to data in Azure Storage](../common/authorize-data-access.md)
- [Assign an Azure role for access to file data](assign-azure-role-data-access.md)
