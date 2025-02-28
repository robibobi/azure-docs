---
title: Create Azure Virtual Desktop host pool - Azure
description: How to create a host pool in Azure Virtual Desktop with PowerShell or the Azure CLI.
author: Heidilohr
ms.topic: how-to
ms.date: 07/23/2021
ms.author: helohr 
ms.custom: devx-track-azurepowershell, devx-track-azurecli

manager: femila
---
# Create an Azure Virtual Desktop host pool with PowerShell or the Azure CLI

>[!IMPORTANT]
>This content applies to Azure Virtual Desktop with Azure Resource Manager Azure Virtual Desktop objects. If you're using Azure Virtual Desktop (classic) without Azure Resource Manager objects, see [this article](./virtual-desktop-fall-2019/create-host-pools-powershell-2019.md).

Host pools are a collection of one or more identical virtual machines within Azure Virtual Desktop. Each host pool can be associated with multiple RemoteApp groups, one desktop app group, and multiple session hosts.

You can create host pools in the following Azure regions:

- Australia East
- Canada Central
- Canada East
- Central US
- East US
- East US 2
- Japan East
- North Central US
- North Europe
- South Central US
- UK South
- UK West
- West Central US
- West Europe
- West US
- West US 2

## Create a host pool

### [Azure PowerShell](#tab/azure-powershell)

If you haven't already done so, follow the instructions in [Set up the PowerShell module](powershell-module.md).

Run the following cmdlet to sign in to the Azure Virtual Desktop environment:

```powershell
New-AzWvdHostPool -ResourceGroupName <resourcegroupname> -Name <hostpoolname> -WorkspaceName <workspacename> -HostPoolType <Pooled|Personal> -LoadBalancerType <BreadthFirst|DepthFirst|Persistent> -Location <region> -DesktopAppGroupName <appgroupname> -PreferredAppGroupType <appgrouptype>
```

This cmdlet will create the host pool, workspace and desktop app group. Additionally, it will register the desktop app group to the workspace. You can either create a workspace with this cmdlet or use an existing workspace.

Run the next cmdlet to create a registration token to authorize a session host to join the host pool and save it to a new file on your local computer. You can specify how long the registration token is valid by using the *-ExpirationTime* parameter.

>[!NOTE]
>The token's expiration date can be no less than an hour and no more than one month. If you set *-ExpirationTime* outside of that limit, the cmdlet won't create the token.

```powershell
New-AzWvdRegistrationInfo -ResourceGroupName <resourcegroupname> -HostPoolName <hostpoolname> -ExpirationTime $((get-date).ToUniversalTime().AddDays(1).ToString('yyyy-MM-ddTHH:mm:ss.fffffffZ'))
```

For example, if you want to create a token that expires in two hours, run this cmdlet:

```powershell
New-AzWvdRegistrationInfo -ResourceGroupName <resourcegroupname> -HostPoolName <hostpoolname> -ExpirationTime $((get-date).ToUniversalTime().AddHours(2).ToString('yyyy-MM-ddTHH:mm:ss.fffffffZ'))
```

After that, run this cmdlet to add Azure Active Directory users to the default desktop app group for the host pool.

```powershell
New-AzRoleAssignment -SignInName <userupn> -RoleDefinitionName "Desktop Virtualization User" -ResourceName <hostpoolname+"-DAG"> -ResourceGroupName <resourcegroupname> -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
```

Run this next cmdlet to add Azure Active Directory user groups to the default desktop app group for the host pool:

```powershell
New-AzRoleAssignment -ObjectId <usergroupobjectid> -RoleDefinitionName "Desktop Virtualization User" -ResourceName <hostpoolname+"-DAG"> -ResourceGroupName <resourcegroupname> -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
```

Run the following cmdlet to export the registration token to a variable, which you will use later in [Register the virtual machines to the Azure Virtual Desktop host pool](#register-the-virtual-machines-to-the-azure-virtual-desktop-host-pool).

```powershell
$token = Get-AzWvdRegistrationInfo -ResourceGroupName <resourcegroupname> -HostPoolName <hostpoolname>
```

### [Azure CLI](#tab/azure-cli)

If you haven't already done so, prepare your environment for the Azure CLI:

[!INCLUDE [azure-cli-prepare-your-environment-no-header.md](../../includes/azure-cli-prepare-your-environment-no-header.md)]

After you sign in, use the [az desktopvirtualization hostpool create](/cli/azure/desktopvirtualization#az-desktopvirtualization-hostpool-create) command to create the new host pool, optionally creating a registration token for session hosts to join the host pool:

```azurecli
az desktopvirtualization hostpool create --name "MyHostPool" \
   --resource-group "MyResourceGroup" \
   --location "MyLocation" \
   --host-pool-type "Pooled" \
   --load-balancer-type "BreadthFirst" \
   --max-session-limit 999 \
   --personal-desktop-assignment-type "Automatic" \
   --registration-info expiration-time="2022-03-22T14:01:54.9571247Z" registration-token-operation="Update" \
   --sso-context "KeyVaultPath" \
   --description "Description of this host pool" \
   --friendly-name "Friendly name of this host pool" \
   --tags tag1="value1" tag2="value2"
```

---

## Create virtual machines for the host pool

Now you can create an Azure virtual machine that can be joined to your Azure Virtual Desktop host pool.

You can create a virtual machine in multiple ways:

- [Create a virtual machine from an Azure Gallery image](../virtual-machines/windows/quick-create-portal.md#create-virtual-machine)
- [Create a virtual machine from a managed image](../virtual-machines/windows/create-vm-generalized-managed.md)
- [Create a virtual machine from an unmanaged image](https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.compute/vm-from-user-image)

>[!NOTE]
>If you're deploying a virtual machine using Windows 7 as the host OS, the creation and deployment process will be a little different. For more details, see [Deploy a Windows 7 virtual machine on Azure Virtual Desktop](./virtual-desktop-fall-2019/deploy-windows-7-virtual-machine.md).

After you've created your session host virtual machines, [apply a Windows license to a session host VM](apply-windows-license.md#manually-apply-a-windows-license-to-a-windows-client-session-host-vm) to run your Windows or Windows Server virtual machines without paying for another license.

## Prepare the virtual machines for Azure Virtual Desktop agent installations

You need to do the following things to prepare your virtual machines before you can install the Azure Virtual Desktop agents and register the virtual machines to your Azure Virtual Desktop host pool:

- You must domain-join the machine. This allows incoming Azure Virtual Desktop users to be mapped from their Azure Active Directory account to their Active Directory account and be successfully allowed access to the virtual machine.
- You must install the Remote Desktop Session Host (RDSH) role if the virtual machine is running a Windows Server OS. The RDSH role allows the Azure Virtual Desktop agents to install properly.

To successfully domain-join, do the following things on each virtual machine:

1. [Connect to the virtual machine](../virtual-machines/windows/quick-create-portal.md#connect-to-virtual-machine) with the credentials you provided when creating the virtual machine.
2. On the virtual machine, launch **Control Panel** and select **System**.
3. Select **Computer name**, select **Change settings**, and then select **Change…**
4. Select **Domain** and then enter the Active Directory domain on the virtual network.
5. Authenticate with a domain account that has privileges to domain-join machines.

    >[!NOTE]
    > If you're joining your VMs to an Azure Active Directory Domain Services (Azure AD DS) environment, ensure that your domain join user is also a member of the [AAD DC Administrators group](../active-directory-domain-services/tutorial-create-instance-advanced.md#configure-an-administrative-group).

>[!IMPORTANT]
>We recommend that you don't enable any policies or configurations that disable Windows Installer. If you disable Windows Installer, the service won't be able to install agent updates on your session hosts, and your session hosts won't function properly.

## Register the virtual machines to the Azure Virtual Desktop host pool

Registering the virtual machines to a Azure Virtual Desktop host pool is as simple as installing the Azure Virtual Desktop agents.

To register the Azure Virtual Desktop agents, do the following on each virtual machine:

1. [Connect to the virtual machine](../virtual-machines/windows/quick-create-portal.md#connect-to-virtual-machine) with the credentials you provided when creating the virtual machine.
2. Download and install the Azure Virtual Desktop Agent.
   - Download the [Azure Virtual Desktop Agent](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrmXv).
   - Run the installer. When the installer asks you for the registration token, enter the value you got from the **Get-AzWvdRegistrationInfo** cmdlet.
3. Download and install the Azure Virtual Desktop Agent Bootloader.
   - Download the [Azure Virtual Desktop Agent Bootloader](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrxrH).
   - Run the installer.

>[!IMPORTANT]
>To help secure your Azure Virtual Desktop environment in Azure, we recommend you don't open inbound port 3389 on your VMs. Azure Virtual Desktop doesn't require an open inbound port 3389 for users to access the host pool's VMs. If you must open port 3389 for troubleshooting purposes, we recommend you use [just-in-time VM access](../security-center/security-center-just-in-time.md). We also recommend you don't assign your VMs to a public IP.

## Update the agent

You'll need to update the agent if you're in one of the following situations:

- You want to migrate a previously registered session host to a new host pool
- The session host doesn't appear in your host pool after an update

To update the agent:

1. Sign in to the VM as an administrator.
2. Go to **Services**, then stop the **Rdagent** and **Remote Desktop Agent Loader** processes.
3. Next, find the agent and bootloader MSIs. They'll either be located in the **C:\DeployAgent** folder or whichever location you saved it to when you installed it.
4. Find the following files and uninstall them:
     
     - Microsoft.RDInfra.RDAgent.Installer-x64-verx.x.x
     - Microsoft.RDInfra.RDAgentBootLoader.Installer-x64

   To uninstall these files, right-click on each file name, then select **Uninstall**.
5. Optionally, you can also remove the following registry settings:
     
     - Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\RDInfraAgent
     - Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\RDAgentBootLoader

6. Once you've uninstalled these items, this should remove all associations with the old host pool. If you want to reregister this host to the service, follow the instructions in [Register the virtual machines to the Azure Virtual Desktop host pool](create-host-pools-powershell.md#register-the-virtual-machines-to-the-azure-virtual-desktop-host-pool).


## Next steps

Now that you've made a host pool, you can populate it with RemoteApps. To learn more about how to manage apps in Azure Virtual Desktop, see the Manage app groups tutorial.

> [!div class="nextstepaction"]
> [Manage app groups tutorial](./manage-app-groups.md)
