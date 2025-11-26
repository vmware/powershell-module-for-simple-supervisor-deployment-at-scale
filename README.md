# Single-Node vSphere Supervisor Deployment Automation

## Overview

This PowerShell automation script is designed to streamline the deployment of a single-node vSphere Supervisor in VMware Cloud Foundation (VCF) 9.x environments. It automates critical steps from initial setup to the creation of the supervisor, including network configuration and content library verification. The script leverages VCF.PowerCLI cmdlets and relies on pre-configured JSON input files (`infrastructure.json` and `supervisor.json`) to tailor the deployment to your environment. Additionally, it addresses the prerequisites for integrating Argo CD operator services and ensures the necessary CLI tools are available for a comprehensive, end-to-end automated solution.

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

Open `infrastructure.json` and review all the fields, updating as required for your environment using the table below as reference. **Accuracy here is crucial for successful deployment.**

#### infrastructure.json Configuration Reference

| Element Name | Default Value May Be Used? | Created by Script? | Notes |
|--------------|----------------------------|-------------------|-------|
| common.vCenterName | No | No | Must be vCenter 9.0 or higher. Script execution system must have HTTPS access to it. |
| common.vCenterUser | Yes | No | While `administrator@vsphere.local` can be used, it's recommended that one use a less privileged user that can create clusters, port groups, and supervisors. SSO users may be used as well. |
| common.EsxHost | No | No | The script execution system must have HTTPS access to the ESX host. |
| common.EsxUser | Yes | No | ESX User must have permission to create a datastore. |
| common.datacenterName | No | No | The Edge Cluster will be created under this virtual datacenter. It is assumed the vSphere datacenter already existed before running the script. |
| common.clusterName | Yes | Yes | The script will create an Edge ESX cluster using the cluster name. |
| common.supervisorName | Yes | Yes | The script will create a vSphere Supervisor using this name. |
| common.argoCD.argoCdOperatorYamlPath | No | No | Path to the ArgoCD operator YAML file. This file is part of the ZIP file download from step #2 or can be downloaded from the Broadcom support portal. For Windows systems, backslashes must be escaped. For MacOS and Linux, there is no need to escape the forward slash. |
| common.argoCD.argoCdDeploymentYamlPath | No | No | Path to the modified ArgoCD instance YAML config. Refer to Step #3. For Windows systems, backslashes must be escaped. For MacOS and Linux, there is no need to escape the forward slash. Namespace in this file must match the namespace in common.argoCD.nameSpace in infrastructure.json. |
| common.argoCD.contextName | Yes | Yes | This is the VCF context name. VCF CLI will configure a new set of VCF contexts to access and configure ArgoCD. |
| common.argoCD.nameSpace | Yes | Yes | This namespace definition must match the one in the yaml file referenced in common.argoCD.argoCdDeploymentYamlPath. |
| common.argoCD.vmClass | Yes | No | This is a list of VM classes already available to the Supervisor that you wish to associate with the ArgoCD namespace. |
| common.datastore.datastoreName | Yes | Yes | The script will prompt the user to select an unformatted disk to format as VMFS using this name. |
| common.storagePolicy.storagePolicyTagCatalog | Yes | Yes | Any TagCatalog name. Script will search if TagCatalog already exists, if not, create a new one. |
| common.storagePolicy.storagePolicyName | Yes | Yes | Policy used to map VMFS datastore specified to the Supervisor instance. |
| common.storagePolicy.storagePolicyType | Yes | No | The only valid value is VMFS. Please do not change otherwise the script will error with a warning to revert the setting to VMFS. |
| common.storagePolicy.storagePolicyRule | Yes | No | The only valid value is "Fully initialized". Please do not change otherwise the script will error with a warning to revert the setting to "Fully initialized". |
| common.virtualDistributedSwitch.vdsName | Yes | Yes | The virtual distributed switch created by the script. |
| common.virtualDistributedSwitch.vdsVersion | Yes | Yes | This must be a valid virtual distributed switch version. The script will create a new VDS if there isn't one already. Leave the value as "9.0.0". |
| common.virtualDistributedSwitch.numUplinks | Yes | Yes | The number of uplinks used by the virtual distributed switch created by the script. |
| common.virtualDistributedSwitch.nicList | Yes | Yes | An array of physical NICs on the ESX host that you wish to attach to the virtual distributed switch created by the script. Script will not migrate vmkernel to the newly created VDS. |
| common.portGroups | Yes | Yes | The names and VLANs of the port groups that you wish to create. Networks referenced for the flbManagementNetwork, flbVirtualServerNetwork, tkgsMgmtNetwork, and tkgsPrimaryWorkloadNetwork in the supervisor.json must exist in this file. The network names must be lower-cased and conform to RFC1123. |

### 5. Configure supervisor.json

Open `supervisor.json` and review all the fields, updating as required for your environment using the table below as reference.

#### supervisor.json Configuration Reference

| Element Name | Default Value May Be Used? | Created by Script? | Notes |
|--------------|----------------------------|-------------------|-------|
| supervisorSpec.controlPlaneVMCount | Yes | No | The value may be either "1" or "3". |
| supervisorSpec.controlPlaneSize | Yes | No | The value may be "Tiny", "Small", "Medium", or "Large". |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbName | Yes | Yes | Foundation Load Balancer Name. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbSize | Yes | No | The value may be "Small", "Medium", "Large", or "X-LARGE". |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbAvailability | Yes | No | The value may be "SINGLE_NODE" or "ACTIVE_PASSIVE" (for two-node deployments). |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVipStartIP | No | Yes | The starting IP for your Foundation Load Balancer virtual IP range. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVipIPCount | No | Yes | The number of IPs used by the Foundation Load Balancer starting from the flbVipStartIP. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbProvider | Yes | Yes | The only valid value is "VSPHERE_FOUNDATION". Please do not change otherwise the script will error. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbDnsServers | No | No | The DNS server(s) for the Foundation Load Balancer network. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbNtpServers | No | No | The NTP server(s) for the Foundation Load Balancer network. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbSearchDomains | No | No | The domain search domain for the Foundation Load Balancer network. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkIpAssignmentMode | Yes | No | The only valid value is "STATIC". Please do not change otherwise the script will error. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkName | Yes | Yes | This network name may be changed, but it must also be changed in infrastructure.json. The network name must be lower-cased and conform to RFC1123. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkType | Yes | No | The only valid value is "DVPG". Please do not change otherwise the script will error. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkIpAddressStartingIp | No | Yes | The starting IP for your Foundation Load Balancer management IP range. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkIpAddressCount | No | Yes | The number of IPs used by the Foundation Load Balancer management network, starting from the flbNetworkIpAddressStartingIp. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbManagementNetwork.flbNetworkGateway | No | Yes | Default gateway for Foundation Load Balancer management network. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkIpAssignmentMode | Yes | No | The only valid value is "STATIC". Please do not change otherwise the script will error. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkName | Yes | Yes | This network name may be changed, but it must also be changed in infrastructure.json common.portGroup.name. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkType | Yes | Yes | The only valid value is "DVPG". Please do not change otherwise the script will error. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkIpAddressStartingIp | No | Yes | The starting IP for your Foundation Load Balancer server network IP range. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkIpAddressCount | Yes | Yes | The number of IPs used by the Foundation Load Balancer virtual server network in the virtual IP range, starting from the flbNetworkIpAddressStartingIp. |
| tkgsComponentSpec.foundationLoadBalancerComponents.flbVirtualServerNetwork.flbNetworkGateway | No | Yes | Default gateway for Foundation Load Balancer server network. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtIpAssignmentMode | Yes | No | The only valid value is "STATIC". Please do not change otherwise the script will error. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkName | Yes | Yes | This network name may be changed, but it must also be changed in infrastructure.json. The network name must be lower-cased and conform to RFC1123. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkGatewayCidr | No | Yes | Default gateway for VKS server network. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkStartingIp | No | Yes | The starting IP for your VKS management network IP range. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkIPCount | No | Yes | The number of IPs used by the Foundation Load Balancer virtual server network in the virtual IP range, starting from the tkgsMgmtNetworkIPCount. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkDnsServers | No | No | The DNS server(s) for the VKS management network. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkSearchDomains | No | No | The domain search domain for the VKS management network. |
| tkgsComponentSpec.tkgsMgmtNetworkSpec.tkgsMgmtNetworkNtpServers | No | No | The NTP server(s) for the VKS management network. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadIpAssignmentMode | Yes | Yes | The only valid value is "STATIC". Please do not change otherwise the script will error. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadNetworkName | Yes | Yes | This network name may be changed, but it must also be changed in infrastructure.json. The network name must be lower-cased and conform to RFC1123. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadNetworkGatewayCidr | No | Yes | Default gateway for VKS workload network. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadNetworkStartingIp | No | Yes | The starting IP address for the VKS workload network virtual IP range. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadNetworkIPCount | No | Yes | The starting IP address for the VKS workload network virtual IP range. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsPrimaryWorkloadNetworkSearchDomains | No | No | The DNS search domain for the VKS workload network. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsWorkloadDnsServers | No | No | The DNS server(s) for the VKS workload network. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsWorkloadNtpServers | No | No | The NTP server(s) for the VKS workload network. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsWorkloadServiceStartIp | No | Yes | The starting IP address for the VKS workload network virtual IP range. |
| tkgsComponentSpec.tkgsPrimaryWorkloadNetwork.tkgsWorkloadServiceCount | Yes | Yes | The maximum IP count, provisioned starting from tkgsWorkloadServiceStartIp IP address, for the VKS workload network virtual IP range. This IP count must fully occupy a CIDR address range, for example, 256 or 512, but not 200 or 500. |

### 6. Argo CD Operator YAML Download (Optional)

Download the necessary YAML file for Argo CD operator creation by following the instructions in the provided documentation.

### 7. Install vcf-cli Plugin

- Follow the installation guide at the official VMware documentation site
- **Important:** After installation, rename file `vcf.exe` on Windows or `vcf` on MacOS or Linux

### 8. Run the Automation Script

Execute the automation script to configure the edge cluster:

```powershell
PS /> OneNodeDeployment.ps1 -infrastructureJson /path/to/infrastructure.json -supervisorJson /path/to/supervisor.json
```

## Important Notes

- **Network Configuration**: The automation currently creates a supervisor with four networks: one for management, one for workload, and two for load balancer. The `infrastructure.json` file expects four VLAN IDs for the virtual distributed switch section accordingly. While there are options to reduce network usage by reusing resource pools, this script adheres to the four-network design as per the linked design documents.

- **CLI Plugin Availability**: For a fully automated end-to-end process, it is required that VCF-CLI and KUBECTL are available on your testbed *before* running this script. Refer to step 7 for VCF CLI installation. Kubectl can be downloaded from upstream at: https://kubernetes.io/docs/tasks/tools/

- **Supported Environment**: This script has been tested and validated on the Mac and Windows platforms. Ensure your execution environment matches this specification for optimal performance and compatibility.

