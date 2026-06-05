# HelloID-Conn-SA-Sync-EntraID-Groups-To-SelfService-Products

> [!IMPORTANT]
> This repository contains the connector and configuration code only. The implementer is responsible to acquire the connection details such as username, password, certificate, etc. You might even need to sign a contract or agreement with the supplier before implementing this connector. Please contact the client's application manager to coordinate the connector requirements.

> [!WARNING]
> **Version 4.0.0 contains breaking changes!**
> 
> This version introduces a complete restructuring of the configuration approach. Do not blindly upgrade from version 3.x or earlier without reviewing and migrating your configuration!
> 
> **Major breaking changes:**
> - Configuration variables have been renamed (e.g., `$ProductSkuPrefix` → `$productIdentifierPrefix`)
> - Product configuration now uses a function-based approach (`New-HelloIDProductConfiguration`)
> - Resource owner mode changed from `$calculateProductResourceOwnerPrefixSuffix` to `$resourceOwnerMode` with "Fixed" or "Calculated" values
> - Update behavior settings have been restructured with new safety controls
> - Certificate-based authentication is now required (Client Secret authentication removed)
> 
> **Before upgrading:**
> 1. Review the CHANGELOG.md for all changes
> 2. Back up your current configuration
> 3. Test the new version in a non-production environment first
> 4. Migrate your configuration according to the new structure
> 
> See [Configuration options](#configuration-options) below for the new configuration structure.

## Table of Contents

- [HelloID-Conn-SA-Sync-EntraID-Groups-To-SelfService-Products](#helloid-conn-sa-sync-entraid-groups-to-selfservice-products)
  - [Table of Contents](#table-of-contents)
  - [Description](#description)
  - [Getting started](#getting-started)
    - [Requirements](#requirements)
    - [App Registration \& Certificate Setup](#app-registration--certificate-setup)
    - [HelloID-specific configuration](#helloid-specific-configuration)
    - [Convert .pfx to base64 string](#convert-pfx-to-base64-string)
    - [Connection settings](#connection-settings)
    - [Configuration options](#configuration-options)
      - [Script Behavior](#script-behavior)
      - [Product Lifecycle](#product-lifecycle)
      - [Source Data Selection](#source-data-selection)
      - [Product Identification](#product-identification)
      - [Product Configuration](#product-configuration)
      - [Resource Owner Configuration](#resource-owner-configuration)
      - [Update Behavior](#update-behavior)
  - [Remarks](#remarks)
    - [Products are created and removed automatically](#products-are-created-and-removed-automatically)
    - [Function-based product configuration](#function-based-product-configuration)
    - [Combined with permissions sync](#combined-with-permissions-sync)
    - [Test run mode](#test-run-mode)
  - [Getting help](#getting-help)
  - [HelloID docs](#helloid-docs)

## Description

HelloID-Conn-SA-Sync-EntraID-Groups-To-SelfService-Products is a scheduled task designed for use with HelloID Service Automation (SA). This task automatically synchronizes Entra ID (Azure AD) groups to HelloID Self Service products, enabling users to request access to groups through the HelloID catalog.

By using this scheduled task, you will have the ability to:

1. Automatically create HelloID Self Service products for each Entra ID group in scope
2. Update product properties when group details change (e.g., product name/description when group displayName changes)
3. Remove or disable products when groups are no longer in scope
4. Configure product actions that grant group membership when products are requested

This eliminates the need to manually create and maintain products for each group, especially valuable in organizations with many Entra ID groups. The task is designed to work in combination with the [Entra ID Group Memberships to Product Assignments Sync](https://github.com/Tools4everBV/HelloID-Conn-SA-Sync-AzureActiveDirectory-Groupmemberships-To-SelfService-Productassignments).

## Getting started

### Requirements

- Windows PowerShell 5.1 installed on the server where the HelloID agent and Service Automation agent are running
- **Not supported** with Cloud Agent (must run On-Premises)
- An App Registration in Microsoft Entra ID configured with certificate-based authentication
- The synchronization must be configured to meet your requirements before scheduling

### App Registration & Certificate Setup

Before implementing this scheduled task, you must configure a Microsoft Entra ID App Registration. During the setup process, you'll create a new App Registration in the Entra portal, assign the necessary API permissions, and generate and assign a certificate.

Follow the official Microsoft documentation for creating an App Registration and setting up certificate-based authentication:

- [App-only authentication with certificate (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/auth-register-app-v2)

### HelloID-specific configuration

Once you have completed the Microsoft setup and followed their best practices, configure the following HelloID-specific requirements.

**API Permissions (Application permissions):**

- `Group.Read.All` - To read group information
- `GroupMember.ReadWrite.All` - To add/remove users from groups

**Certificate:**

- Upload the public key file (.cer) in Entra ID
- Provide the certificate as a Base64 string in HelloID

> [!NOTE]
> For more information about the required permissions, please see the Microsoft docs:
> - [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)
> - [Find the permissions required to run any Microsoft Graph cmdlet](https://learn.microsoft.com/en-us/graph/permissions-reference)
> - [View and assign administrator roles in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/manage-roles-portal)

### Convert .pfx to base64 string

HelloID requires a base64 string to import the certificate. Use the example below to create a base64 string:

```powershell
$filePath = 'C:\Cert'
$pfxCertName = 'Cert.pfx'
$pfxPath = "$filePath\$pfxCertName"

$fileContentBytes = [System.IO.File]::ReadAllBytes("$pfxPath")
[System.Convert]::ToBase64String($fileContentBytes) | Set-Content "$filePath\HelloID_Cert_Base64.txt"
```

### Connection settings

The following global variables must be configured in HelloID when setting up the scheduled task.

| Variable name | Description | Required |
| ------------- | ----------- | -------- |
| portalBaseUrl | The base URL of your HelloID portal | Yes (Default Global Variable) |
| portalApiKey | The API key for HelloID portal access | Yes (Default Global Variable) |
| portalApiSecret | The API secret for HelloID portal access | Yes (Default Global Variable) |
| EntraIdTenantId | The Entra tenant ID (directory ID) | Yes (Recommended as Global Variable) |
| EntraIdAppId | The unique identifier (ID) of the App Registration in Microsoft Entra ID | Yes (Recommended as Global Variable) |
| EntraIdCertificateBase64String | The Base64-encoded string representation of the app certificate | Yes (Recommended as Global Variable) |
| EntraIdCertificatePassword | The password associated with the app certificate | Yes (Recommended as Global Variable) |

> [!NOTE]
> When running inside HelloID, `portalBaseUrl`, `portalApiKey`, and `portalApiSecret` are provided automatically as default global variables.

### Configuration options

The scheduled task includes extensive configuration options organized into logical sections. All configuration is done directly in the PowerShell script through variables and functions. Below is an overview of the main configuration areas.

#### Script Behavior

Controls testing and logging behavior:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $dryRun | If $true, shows what would happen without making changes | $false |
| $verboseLogging | If $true, logs every action (generates lots of log data) | $false |
| $testRun | If $true, limits operations based on max values below | $true |
| $testRunMaxCreates | Maximum products to CREATE in test run | 1 |
| $testRunMaxUpdates | Maximum products to UPDATE in test run | 1 |
| $testRunMaxDeletes | Maximum products to DELETE in test run | 1 |

#### Product Lifecycle

Safety thresholds and removal behavior:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $createThreshold | Maximum number of NEW products to create in one run (safety limit) | 10 |
| $updateThreshold | Maximum number of EXISTING products to update in one run | 10 |
| $removeThreshold | Maximum number of products to disable/remove in one run | 10 |
| $removeProductBehavior | What happens when a product no longer exists: "None", "Disable", or "Remove" | "Remove" |
| $removeResourceOwnerGroupWithProduct | Remove resource owner group when product is removed | $true |

#### Source Data Selection

Controls which Entra ID groups are synchronized:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $MSGraphBaseUri | Microsoft Graph base URI | "https://graph.microsoft.com/" |
| $entraIDGroupPropertiesToRetrieve | Array of group properties to retrieve | See script for full list |
| $entraIDGroupsSearchFilter | Filter which groups to sync (e.g., "`$search=`"displayName:IT_`"") | $null (all groups) |

#### Product Identification

Defines how products are uniquely identified:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $productIdentifierPrefix | Prefix used in product codes (max 8 characters, UPPERCASE, no dashes) | "ENTRAGRP" |
| $sourceObjectUniqueProperty | Source object property used to uniquely identify objects | "id" |

#### Product Configuration

The `New-HelloIDProductConfiguration` function defines all product properties. This function is called once per group and returns a hashtable containing:

- **Identification & Naming**: Code, Name, Description, SourceIdentifier
- **Visibility & Access**: Visibility, AccessGroups
- **Request Settings**: RequestCommentOption, AllowMultipleRequests
- **Approval & Workflow**: ApprovalWorkflowId
- **Appearance**: UseFaIcon, FaIcon, Icon
- **Category**: Category name
- **Form**: FormId (optional)
- **Lifecycle**: ReturnOnUserDisable
- **Time Limits**: HasTimeLimit, ManagerCanOverrideDuration, LimitType, OwnershipMaxDuration
- **Agent Pool**: AgentPool name
- **Pricing**: ShowPrice, Price (optional)
- **Risk Assessment**: HasRiskFactor, RiskFactor (optional)
- **Limits**: MaxCount (optional)
- **Product Actions**: onRequest, onApprove, onDeny, onReturn, onWithdrawn

> [!TIP]
> The function includes comprehensive inline documentation explaining each property. Review the script to customize product configuration to your requirements.

#### Resource Owner Configuration

Determines who manages the products:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $resourceOwnerMode | "Fixed" (same group for all) or "Calculated" (unique per product) | "Calculated" |
| $productResourceOwner | Resource owner group when using Fixed mode | "Local/__HelloID_Administrators" |
| $calculatedResourceOwnerGroupSource | Source for calculated groups: "AzureAD" or "Local" | "Local" |
| $calculatedResourceOwnerGroupPrefix | Prefix for calculated resource owner group names | "" |
| $calculatedResourceOwnerGroupSuffix | Suffix for calculated resource owner group names | " Resource Owner" |

#### Update Behavior

Controls when and how existing products are updated:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| $overwriteExistingProduct | If $true, updates existing products with configured properties | $true |
| $productPropertiesToUpdate | Array of product property names to update | @("name") |
| $updateResourceOwnerGroupOnNameChange | Rename resource owner groups when source names change (Calculated mode only) | $true |
| $accessGroupUpdateBehavior | How to update access groups: "None", "Add", or "Replace" | "None" |
| $overwriteExistingProductAction | If $true, overwrites existing product action scripts | $false |
| $addMissingProductAction | If $true, adds new actions to existing products | $false |
| $removeUnconfiguredActions | If $true, removes actions not in $actionsToUpdate | $false |
| $actionsToUpdate | Hashtable specifying which actions to update | See script |

> [!WARNING]
> - Setting `$overwriteExistingProduct` to `$true` will update ALL products matching the filter
> - Overwriting or removing is IRREVERSIBLE - make backups before enabling
> - Set update variables back to `$false` after bulk updates to prevent unintended changes

## Remarks

### Products are created and removed automatically

By default, this scheduled task both creates AND removes (or disables) products automatically. The `$removeProductBehavior` setting controls what happens to products when groups are no longer in scope:

- `"None"`: Keep products even when they no longer exist (requires manual cleanup)
- `"Disable"`: Disable products when they no longer exist (reversible, recommended for testing)
- `"Remove"`: Permanently delete products when they no longer exist (irreversible)

> [!WARNING]
> Make sure your configuration is correct before running in production, especially the `$removeProductBehavior` setting. Use `$testRun = $true` and `$dryRun = $true` for initial testing.

### Function-based product configuration

Version 4.0.0 introduces a function-based approach to product configuration. Instead of configuring properties through multiple script variables, all product settings are now defined in the `New-HelloIDProductConfiguration` function. This provides:

- **Better organization**: All product properties in one place
- **Comprehensive documentation**: Every property includes detailed inline comments
- **Flexibility**: Easily customize products per group using PowerShell logic
- **Maintainability**: Changes to product structure are easier to implement

### Combined with permissions sync

This scheduled task is designed to work in combination with the [Entra ID Group Memberships to Product Assignments Sync](https://github.com/Tools4everBV/HelloID-Conn-SA-Sync-AzureActiveDirectory-Groupmemberships-To-SelfService-Productassignments). Together, these tasks provide:

1. **This task**: Creates products for Entra ID groups with grant/revoke actions
2. **Permissions sync**: Automatically assigns products to users who already have group membership

This combination ensures that:

- Users who already have access receive the product automatically (showing existing access)
- Users can request access through Self Service (new access requests)
- Revoking the product removes the group membership

### Test run mode

The scheduled task includes a test run mode to limit operations during testing:

```powershell
$testRun = $true
$testRunMaxCreates = 1  # Only create 1 product
$testRunMaxUpdates = 1  # Only update 1 product
$testRunMaxDeletes = 1  # Only delete/disable 1 product
```

When `$testRun = $true`, the task will still retrieve ALL groups for correct comparison, but will limit the actual create/update/delete operations. This allows you to safely test the synchronization behavior without affecting all products.

> [!TIP]
> Set `$testRun = $false` only after you've verified the task works correctly with test limits enabled.

## Getting help

> [!TIP]
> For more information on how to configure a HelloID PowerShell scheduled task, please refer to our [documentation](https://docs.helloid.com/en/service-automation/scheduled-tasks.html) pages.

## HelloID docs

The official HelloID documentation can be found at: https://docs.helloid.com/
