# Simple Supervisor Deployment at Scale Automation

[![PowerShell](https://img.shields.io/badge/PowerShell-7.2%2B-blue.svg)](https://github.com/PowerShell/PowerShell)
[![License](https://img.shields.io/badge/License-Broadcom-green.svg)](LICENSE.md)
[![Version](https://img.shields.io/badge/Version-1.0.0.2-orange.svg)](CHANGELOG.md)
[![GitHub Clones](https://img.shields.io/badge/dynamic/json?color=success&label=Clone&query=count&url=https://gist.githubusercontent.com/nathanthaler/7c5ed25bb9cea6eef7f015be50e44a6f/raw/clone.json&logo=github)](https://gist.githubusercontent.com/nathanthaler/7c5ed25bb9cea6eef7f015be50e44a6f/raw/clone.json)

## Overview

The **SimpleSupervisorDeploymentAtScale** PowerShell module automates the end-to-end deployment of a vSphere Supervisor in VMware Cloud Foundation (VCF) 9.x environments based on this [design guidance](https://blogs.vmware.com/cloud-foundation/2025/07/14/modernizing-your-edge-with-single-node-vsphere-supervisor-in-vmware-cloud-foundation-9-0/). It automates critical steps from initial setup to the creation of the supervisor, including network configuration and content library verification. The script leverages VCF.PowerCLI cmdlets and relies on pre-configured JSON input files (`infrastructure.json` and `supervisor.json`) to tailor the deployment to your environment. Additionally, it addresses the prerequisites for integrating Argo CD operator services and ensures the necessary CLI tools are available for a comprehensive, end-to-end automated solution.

## Pre-requisites

- **VCF 9.x Environment**: A VCF 9.x environment running with a vCenter instance must be available
- **Supervisor Version**: 9.0.0.0100-24845085 or later
- **Host Preparation**: The host is already prepped with ESX and has appropriate network setup
- **Network Connectivity**: ESX and vCenter network connectivity is established
- **Provisioning Access**: Connectivity must be available from the Provisioning host to Supervisor Management Network
- **Datacenter Setup**: Datacenter defined in `infrastructure.json` must already be created and ESX image already populated in the vLCM depot
- **PowerShell**: Version 7.0 or later installed on your system. If not, download and install it from the official Microsoft website
- **kubectl**: Installed on your system. kubectl can be downloaded from upstream at: https://kubernetes.io/docs/tasks/tools/

## Functions Performed by this Automation

This automation script will cover the following key functions:

1. **Cluster Creation and Host Addition**: It will create a new cluster in vCenter and add the specified host to this cluster

2. **Cluster Configuration**:
   - Distributed Resource Scheduler (DRS) will be set to Automatic
   - High Availability (HA) admission control will be disabled

3. **Creation of VDS and related port groups**: A vSphere Distributed Switch (VDS) will be created, along with necessary port groups for edge application and services

4. **Datastore Configuration**:
   - VMFS datastore will be configured based on available disk
   - Local storage only, no external storage

5. **Storage Policy for Edge Datastore**: A storage policy will be created specifically for the Edge Datastore

6. **vSphere Supervisor Enablement**: The vSphere Supervisor feature will be enabled on the newly configured cluster

7. **Supervisor Services**: VM Operator service, VKS Kubernetes Service, Velero backup and restore service, Argo CD Operator Service

8. **Argo CD Instance Creation**: An instance of Argo CD will be created and configured for use

## Installation

### From PowerShell Gallery (Recommended)

```powershell
# Install the module
Install-Module -Name SimpleSupervisorDeploymentAtScale -Scope CurrentUser

# Import the module
Import-Module SimpleSupervisorDeploymentAtScale

# Verify installation
Get-Module -Name SimpleSupervisorDeploymentAtScale
```

### Manual Installation

1. Download the module files from this repository
2. Extract to your PowerShell modules directory:
   - **Windows**: `$env:USERPROFILE\Documents\PowerShell\Modules\SimpleSupervisorDeploymentAtScale\`
   - **Linux/macOS**: `~/.local/share/powershell/Modules/SimpleSupervisorDeploymentAtScale/`
3. Import the module: `Import-Module SimpleSupervisorDeploymentAtScale`

## Quick Start

### 1. Get Template Files

The module includes default configuration templates. Copy them to your working directory:

```powershell
# Copy all template files to current directory
Copy-SimpleSupervisorTemplates

# Or specify a destination
Copy-SimpleSupervisorTemplates -DestinationPath "./config"

# Preview what would be copied (without actually copying)
Copy-SimpleSupervisorTemplates -WhatIf
```

This will copy four template files:
- `infrastructure.json` - Infrastructure configuration (vCenter, cluster, network, storage)
- `supervisor.json` - Supervisor cluster configuration (networks, load balancer)
- `argocd-deployment.yml` - ArgoCD instance deployment YAML
- `1.0.1-24896502.yml` - ArgoCD operator package YAML

### 2. Configure Template Files

Edit the template files with your environment-specific values:

- **infrastructure.json**: Update vCenter details, ESX host, cluster name, network settings, storage configuration
- **supervisor.json**: Configure supervisor cluster settings, network IP ranges, load balancer configuration
- **argocd-deployment.yml**: Configure ArgoCD instance settings
- **1.0.1-24896502.yml**: Typically no changes needed (ArgoCD operator package)

**Need help configuring your JSON files?** Use the built-in helper functions to view detailed configuration reference tables:

```powershell
# View infrastructure.json configuration reference (auto-detects best format)
Show-InfrastructureJsonConfigurationHelp

# View supervisor.json configuration reference (auto-detects best format)
Show-SupervisorJsonConfigurationHelp

# Use list format for narrow screens
Show-InfrastructureJsonConfigurationHelp -Format List
Show-SupervisorJsonConfigurationHelp -Format List

# Use table format for wide screens (with separators for readability)
Show-InfrastructureJsonConfigurationHelp -Format Table
Show-SupervisorJsonConfigurationHelp -Format Table

# Use interactive grid view (Windows PowerShell only)
Show-InfrastructureJsonConfigurationHelp -Format GridView
Show-SupervisorJsonConfigurationHelp -Format GridView
```

**Format Options:**
- **Auto** (default): Automatically selects the best format based on terminal width. Uses 'List' for narrow screens (< 120 characters) and 'Table' for wide screens (≥ 120 characters).
- **List**: Displays each field on its own line. Works best for narrow screens (40-50+ characters). No column wrapping issues.
- **Table**: Displays data in a table format with horizontal separators between rows. Best for wide screens (120+ characters).
- **GridView**: Opens an interactive grid view window with sorting and filtering capabilities. Works on any screen size but requires Windows PowerShell (not available in PowerShell Core on macOS/Linux).

These functions display comprehensive tables showing:
- All configuration elements and their paths
- Whether default values can be used
- Whether elements are created by the script
- Important notes and requirements for each field

## 3. Run Deployment

```powershell
# Deploy using default configuration files
Start-SimpleSupervisorDeploymentAtScale

# Or specify custom file paths
Start-SimpleSupervisorDeploymentAtScale `
    -InfrastructureJson "./config/infrastructure.json" `
    -SupervisorJson "./config/supervisor.json" `
    -LogLevel INFO
```


## Module Functions

### Start-SimpleSupervisorDeploymentAtScale

Main deployment function that automates the complete vSphere Supervisor deployment process.

**Parameters:**
- `InfrastructureJson` (String, optional) - Path to infrastructure configuration JSON file. Default: `"infrastructure.json"`
- `SupervisorJson` (String, optional) - Path to supervisor configuration JSON file. Default: `"supervisor.json"`
- `LogLevel` (String, optional) - Minimum log level for console output. Valid values: `DEBUG`, `INFO`, `ADVISORY`, `WARNING`, `EXCEPTION`, `ERROR`. Default: `"INFO"`
- `Version` (Switch, optional) - Display module version and exit

**Examples:**

```powershell
# Basic deployment with default files
Start-SimpleSupervisorDeploymentAtScale

# Deployment with custom configuration files
Start-SimpleSupervisorDeploymentAtScale `
    -InfrastructureJson "config/prod-infrastructure.json" `
    -SupervisorJson "config/prod-supervisor.json"

# Deployment with DEBUG logging for troubleshooting
Start-SimpleSupervisorDeploymentAtScale -LogLevel DEBUG

# Check module version
Start-SimpleSupervisorDeploymentAtScale -Version
```

### Copy-SimpleSupervisorTemplates

Copies all required template files from the module installation directory to a specified destination.

**Parameters:**
- `DestinationPath` (String, optional) - Destination directory for template files. Default: Current working directory (`$PWD`)
- `WhatIf` (Switch) - Preview what would happen without actually copying files
- `Confirm` (Switch) - Prompt for confirmation before performing operations

**Examples:**

```powershell
# Copy templates to current directory
Copy-SimpleSupervisorTemplates

# Copy templates to specific directory
Copy-SimpleSupervisorTemplates -DestinationPath "./config"

# Preview what would be copied
Copy-SimpleSupervisorTemplates -WhatIf

# Copy with confirmation prompt
Copy-SimpleSupervisorTemplates -DestinationPath "./config" -Confirm
```

**Template Files Included:**
- `infrastructure.json` - Infrastructure configuration template
- `supervisor.json` - Supervisor cluster configuration template
- `argocd-deployment.yml` - ArgoCD deployment YAML template
- `1.0.1-24896502.yml` - ArgoCD operator package template

### Show-InfrastructureJsonConfigurationHelp

Displays a comprehensive reference table for configuring the `infrastructure.json` file. This helper function shows all configuration elements, whether default values can be used, whether elements are created by the script, and important notes for each field.

**Parameters:**
- `Format` (String, optional) - Output format. Valid values: `Auto` (default), `List`, `Table`, `GridView`. Default: `Auto`
- `Filter` (String, optional) - Filters configuration elements by Element Name using wildcard matching. The filter is automatically wrapped with wildcards (*) on both sides.

**Examples:**

```powershell
# Display infrastructure.json configuration reference (auto-detects best format)
Show-InfrastructureJsonConfigurationHelp

# Use list format for narrow screens
Show-InfrastructureJsonConfigurationHelp -Format List

# Use table format for wide screens (with separators)
Show-InfrastructureJsonConfigurationHelp -Format Table

# Use interactive grid view (Windows PowerShell only)
Show-InfrastructureJsonConfigurationHelp -Format GridView

# Filter to show only argoCD related elements
Show-InfrastructureJsonConfigurationHelp -Filter argoCD

# Combine filter with format
Show-InfrastructureJsonConfigurationHelp -Filter storagePolicy -Format List
```

**Format Options:**
- **Auto**: Automatically selects the best format based on terminal width
- **List**: Vertical layout, ideal for narrow screens (40-50+ characters)
- **Table**: Table format with horizontal separators, ideal for wide screens (120+ characters)
- **GridView**: Interactive window with sorting/filtering (Windows PowerShell only)

The output includes detailed information about:
- vCenter and ESX host configuration
- Cluster and datacenter settings
- Storage policy and datastore configuration
- Virtual Distributed Switch (VDS) settings
- Port group and network configuration
- ArgoCD operator and deployment settings

### Show-SupervisorJsonConfigurationHelp

Displays a comprehensive reference table for configuring the `supervisor.json` file. This helper function shows all configuration elements, whether default values can be used, whether elements are created by the script, and important notes for each field.

**Parameters:**
- `Format` (String, optional) - Output format. Valid values: `Auto` (default), `List`, `Table`, `GridView`. Default: `Auto`
- `Filter` (String, optional) - Filters configuration elements by Element Name using wildcard matching. The filter is automatically wrapped with wildcards (*) on both sides.

**Examples:**

```powershell
# Display supervisor.json configuration reference (auto-detects best format)
Show-SupervisorJsonConfigurationHelp

# Use list format for narrow screens
Show-SupervisorJsonConfigurationHelp -Format List

# Use table format for wide screens (with separators)
Show-SupervisorJsonConfigurationHelp -Format Table

# Use interactive grid view (Windows PowerShell only)
Show-SupervisorJsonConfigurationHelp -Format GridView

# Filter to show only tkgsComponentSpec related elements
Show-SupervisorJsonConfigurationHelp -Filter tkgsComponentSpec

# Filter for load balancer elements
Show-SupervisorJsonConfigurationHelp -Filter flb -Format List
```

**Format Options:**
- **Auto**: Automatically selects the best format based on terminal width
- **List**: Vertical layout, ideal for narrow screens (40-50+ characters)
- **Table**: Table format with horizontal separators, ideal for wide screens (120+ characters)
- **GridView**: Interactive window with sorting/filtering (Windows PowerShell only)

The output includes detailed information about:
- Supervisor control plane configuration (VM count, size)
- Foundation Load Balancer settings (name, size, availability)
- Network IP ranges and assignments
- DNS, NTP, and search domain configuration
- VKS management and workload network settings

## Configuration Files

### infrastructure.json

Contains infrastructure-level configuration including:

- **vCenter Connection**: Server name, credentials
- **ESX Host**: Hostname, credentials
- **Cluster Configuration**: Datacenter, cluster name
- **Storage**: Datastore name, storage policy settings
- **Networking**: Virtual Distributed Switch (VDS) configuration, port groups, VLANs
- **ArgoCD**: Operator and deployment YAML paths, namespace, VM classes

**Configuration Help**: Run `Show-InfrastructureJsonConfigurationHelp` to view a detailed reference table with all configuration elements, default value options, and important notes. Use the `-Format` parameter to choose the display format (Auto, List, Table, or GridView).

### supervisor.json

Contains supervisor cluster configuration including:

- **Control Plane**: VM count and size settings
- **Foundation Load Balancer**: Name, size, availability mode, IP ranges
- **Network Configuration**: Management and workload network settings, IP assignments
- **DNS/NTP**: Server and search domain configuration

**Configuration Help**: Run `Show-SupervisorJsonConfigurationHelp` to view a detailed reference table with all configuration elements, default value options, and important notes. Use the `-Format` parameter to choose the display format (Auto, List, Table, or GridView).

# Logging

The module provides comprehensive logging with multiple levels:

- **DEBUG**: Detailed diagnostic information for troubleshooting
- **INFO**: General informational messages about deployment progress
- **ADVISORY**: Important notices that don't indicate problems
- **WARNING**: Warning messages about potential issues
- **EXCEPTION**: Caught exceptions that were handled
- **ERROR**: Error messages indicating failures

Log files are created in the `logs` subdirectory with the naming pattern:
```
logs/SimpleSupervisorDeploymentAtScale-YYYY-MM-DD.log
```

## Troubleshooting

### Template Files Not Found

If `Copy-SimpleSupervisorTemplates` cannot find template files:

```powershell
# Check module installation path
(Get-Module -Name SimpleSupervisorDeploymentAtScale -ListAvailable).ModuleBase

# Verify templates exist
Test-Path "$((Get-Module SimpleSupervisorDeploymentAtScale -ListAvailable).ModuleBase)\Templates"
```

### Module Not Found

If the module is not recognized:

```powershell
# Check if module is installed
Get-Module -Name SimpleSupervisorDeploymentAtScale -ListAvailable

# Check module path
$env:PSModulePath

# Reinstall if needed
Install-Module -Name SimpleSupervisorDeploymentAtScale -Force
```

### Deployment Failures

1. **Enable DEBUG logging**: `Start-SimpleSupervisorDeploymentAtScale -LogLevel DEBUG`
2. **Check log files**: Review `logs/SimpleSupervisorDeploymentAtScale-*.log`
3. **Validate JSON files**: Ensure configuration files are valid JSON
4. **Verify prerequisites**: Confirm VCF.PowerCLI, kubectl, and vcf CLI are available

### Common Issues

- **"Templates directory not found"**: Reinstall the module or verify FileList in manifest
- **"Required template files not found"**: Run `Copy-SimpleSupervisorTemplates` to verify files exist
- **"Unable to determine module installation path"**: Ensure module is properly installed via `Install-Module`

## Requirements

### PowerShell Module Dependencies

- **VCF.PowerCLI** (Version 9.0 or later)
  ```powershell
  Install-Module -Name VCF.PowerCLI -MinimumVersion 9.0
  ```

### External Tools

- **kubectl**: Required for ArgoCD operations
- **vcf CLI**: Required for supervisor management operations

### Environment

- VMware Cloud Foundation 9.x
- vCenter Server with administrative access
- ESX hosts in connected state
- Network connectivity to vCenter and ESX hosts

## Version History

See [CHANGELOG.md](CHANGELOG.md) for detailed version history.

### Version 1.0.0.3 (Current)
- Added `Copy-SimpleSupervisorTemplates` function for template file management
- Added `Show-InfrastructureJsonConfigurationHelp` and `Show-SupervisorJsonConfigurationHelp` helper functions for configuration reference
- Included all template files in module package
- Enhanced path resolution for various installation scenarios
- Improved WhatIf support and user experience

### Version 1.0.0.2
- Significant reliability and performance improvements
- Cross-platform compatibility fixes
- Enhanced error handling and logging

## Contributing

This module is maintained by Broadcom. For issues, feature requests, or contributions, please refer to the project repository.

## License

Copyright (c) 2025 Broadcom. All Rights Reserved.

See the module manifest or LICENSE file for full license details.

## Support

For support and documentation:
- **Project Repository**: [GitHub Repository](https://github.com/vmware/powershell-module-for-simple-supervisor-deployment-at-scale)
- **Documentation**: See module help: `Get-Help Start-SimpleSupervisorDeploymentAtScale -Full`
- **VMware Cloud Foundation Documentation**: [VCF Documentation](https://techdocs.broadcom.com/us/en/vmware-cloud-foundation.html)

## Related Resources

- [VMware Cloud Foundation Documentation](https://techdocs.broadcom.com/us/en/vmware-cloud-foundation.html)
- [Single-Node vSphere Supervisor Design Guidance](https://blogs.vmware.com/cloud-foundation/2025/07/14/modernizing-your-edge-with-single-node-vsphere-supervisor-in-vmware-cloud-foundation-9-0/)
- [ArgoCD Service Documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vsphere-supervisor-services-and-standalone-components/latest/using-supervisor-services/using-argo-cd-service/argocd-custom-resrouce-reference.html)



## Execution Steps

### 1. Install VCF.PowerCLI Module

Install the required PowerCLI module for VCF.

### 2. Download Script Files

Download the provided zip file. The downloaded file has the following structure:

- `OneNodeDeployment.ps1` is the main launcher file
- `1.0.1-24896502.yml` is the ArgoCD Operator YAML file supplied by Broadcom
- Neither file requires any modifications for a standard one node deployment
- `infrastructure.json`, `supervisor.json` and `argocd-deployment.yml` are parameter templates that require updating to align with your edge environment
  - `infrastructure.json` contains vSphere configuration details
  - `supervisor.json` contains Supervisor networking, availability and sizing parameters
  - `argocd-deployment.yml` is the ArgoCD instance YAML that defines ArgoCD resource deployment

### 3. Configure argocd-deployment.yml

Open `argocd-deployment.yml` and populate the fields with the details required to run ArgoCD instances to manage your edge application. Follow ArgoCD Instance configuration details in the provided documentation as reference.

### 4. Configure infrastructure.json

Open `infrastructure.json` and review all the fields, updating as required for your environment. **Accuracy here is crucial for successful deployment.**

**Quick Reference**: Use the built-in helper function to view the configuration reference table:
```powershell
# Auto-detects best format based on terminal width
Show-InfrastructureJsonConfigurationHelp

# Or specify format explicitly
Show-InfrastructureJsonConfigurationHelp -Format List    # For narrow screens
Show-InfrastructureJsonConfigurationHelp -Format Table     # For wide screens
Show-InfrastructureJsonConfigurationHelp -Format GridView # Interactive (Windows only)
```

Alternatively, refer to the table in `Admin.Guide.Single.Node.Supervisor.rtf` for detailed configuration guidance.

### 5. Configure supervisor.json

Open `supervisor.json` and review all the fields, updating as required for your environment.

**Quick Reference**: Use the built-in helper function to view the configuration reference table:
```powershell
# Auto-detects best format based on terminal width
Show-SupervisorJsonConfigurationHelp

# Or specify format explicitly
Show-SupervisorJsonConfigurationHelp -Format List    # For narrow screens
Show-SupervisorJsonConfigurationHelp -Format Table     # For wide screens
Show-SupervisorJsonConfigurationHelp -Format GridView # Interactive (Windows only)
```

Alternatively, refer to the table in `Admin.Guide.Single.Node.Supervisor.rtf` for detailed configuration guidance.

### 6. Argo CD Operator YAML Download (Optional)

Download the necessary YAML file for Argo CD operator creation by following the instructions in the provided documentation.

### 7. Install vcf-cli Plugin

- Follow the installation guide at the official VMware documentation site
- **Important:** After installation, rename file `vcf.exe` on Windows or `vcf` on MacOS or Linux


## Important Notes

- **Network Configuration**: The automation currently creates a supervisor with four networks: one for management, one for workload, and two for load balancer. The `infrastructure.json` file expects four VLAN IDs for the virtual distributed switch section accordingly. While there are options to reduce network usage by reusing resource pools, this script adheres to the four-network design as per the linked design documents.

- **CLI Plugin Availability**: For a fully automated end-to-end process, it is required that VCF-CLI and KUBECTL are available on your testbed *before* running this script. Refer to step 7 for VCF CLI installation. Kubectl can be downloaded from upstream at: https://kubernetes.io/docs/tasks/tools/

- **Supported Environment**: This script has been tested and validated on the Mac and Windows platforms. Ensure your execution environment matches this specification for optimal performance and compatibility.