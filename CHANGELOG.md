# Change Log

All notable changes to this project will be documented in this file. The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [4.0.0] - 05-06-2026

### Added

- **Test Run Mode**: Added test run settings to limit operations per type (creates, updates, deletes) for safer testing
- **Safety Thresholds**: Added configurable thresholds for create, update, and remove operations to prevent accidental mass changes
- **Configurable Remove Behavior**: Added three options for handling products no longer in source system:
  - `None`: Keep products (requires manual cleanup)
  - `Disable`: Disable products (reversible, recommended)
  - `Remove`: Permanently delete products (irreversible)
- **Resource Owner Group Cleanup**: Option to automatically remove resource owner groups when products are removed
- **Product Configuration Function**: New `New-HelloIDProductConfiguration` function with comprehensive inline documentation for all product properties
- **Granular Property Updates**: Added ability to specify exactly which product properties to update on existing products
- **Access Group Update Behavior**: Three modes for updating access groups (None, Add, Replace)
- **Group Property Selection**: Configurable list of group properties to retrieve from Microsoft Graph
- **Category Auto-Creation**: Option to automatically create product categories if they don't exist
- **Product Lifecycle Actions**: Expanded support for all lifecycle actions (onRequest, onApprove, onDeny, onReturn, onWithdrawn)
- **Action Management**: Granular control over updating, adding, and removing product actions
- **Update Resource Owner on Name Change**: Option to rename resource owner groups when source object names change

### Changed

- **BREAKING**: Complete restructuring of configuration with organized sections:
  - Connection Configuration
  - Script Behavior
  - Product Lifecycle
  - Source Data Selection
  - Product Identification
  - Product Configuration Function
  - Resource Owner Configuration
  - Update Behavior
- **BREAKING**: Renamed configuration variables for clarity and consistency:
  - `$ProductSkuPrefix` → `$productIdentifierPrefix`
  - `$entraIDGroupUniqueProperty` → `$sourceObjectUniqueProperty`
  - `$productResourseOwner` → `$productResourceOwner`
  - `$calculateProductResourceOwnerPrefixSuffix` → `$resourceOwnerMode` (now "Fixed" or "Calculated")
  - `$overwriteAccessGroup` → `$accessGroupUpdateBehavior`
  - `$removeProduct` → `$removeProductBehavior`
- **BREAKING**: Product configuration now uses a function-based approach instead of inline variables
- **BREAKING**: Action scripts now referenced by variable name to keep configuration clean
- **BREAKING**: Certificate-based authentication is now required (Client Secret authentication removed)
- Enhanced inline documentation with detailed explanations for every configuration option
- Improved resource owner mode configuration with clearer "Fixed" vs "Calculated" terminology
- Better organization of update behavior settings with clear warnings
- Standardized variable naming conventions throughout the script

### Improved

- Comprehensive inline documentation for all configuration sections
- Clear warnings and recommendations for potentially dangerous operations
- Better structured sections with clear separation of concerns
- More intuitive configuration with examples and best practices
- Enhanced safety with multiple threshold options and test run mode

### Fixed

- Configuration consistency issues with resource owner group management
- Unclear update behavior options now clearly documented

## [3.0.0] - 2024

### Changed

- Authentication changed to certificate-based for improved security
- Updated Microsoft Graph API integration
- Improved error handling and logging

### Fixed

- Various bug fixes and stability improvements

## [2.0.0] - 2023

### Added

- Support for calculated resource owner groups
- Enhanced product configuration options
- Access group management

### Changed

- Improved product lifecycle management
- Enhanced synchronization logic

## [1.0.0] - 2022

This is the first official release of HelloID-Conn-SA-Sync-EntraID-Groups-To-SelfService-Products.

### Added

- Initial implementation of Entra ID Groups to HelloID Self Service Products synchronization
- Automatic product creation based on Entra ID groups
- Product removal when groups are no longer in scope
- Resource owner group support
- Basic product configuration options
- Microsoft Graph API integration for group management
- Product actions for granting and revoking group memberships
