# Azure Windows VM  Using Powershell Cmdlet.
This article will continue the journey. You’ll create the virtual machine itself, then discuss some management techniques for your VM. Here, we shall explain how a windows VM can be created using azure powershell cli commands and the necessary parameters.

**Create a Windows Virtual Machine on Using different Azure Cloud Services.**

1. Azure Portal
2. Azure PowerShell
3. ARM Template
4. Programming, (Terraform, Puppet etc.)


## Creating a VM using Azure portal :

The Management portal provides many images and scripting tools that help you to create  new virtual machine in Azure. The template images that are available in the portal are created and fully supported by either Microsoft or an authorized third party.

**Walkthrough:**


1. Azure portal -> on the Hub Menu, click new-> Computed-> Windows Server 2012 R2 Data center
2. Note :to find additional images,click Marketplace and then search or filter for available items.
3. On the windows server 2012 R2 Data center page, under select a deployment model = Resource Manager ->Create
4. Create a virtual machine blade ->
5. Basics - >provide values for Name, Username & Password, Resource Group ->  Ok
6. Size -> Select an appropriate virtual machine size for your needs. Note the azure recommends size automatically based on the images you select.
7. Settings to see the storage & networking settings for the new virtual machine. For a first virtual machine you can generally accept the default settings.
8. Click create

**Parameters :** 
The parameters which all are necessary in order to create a Virtual Machine is as follows.

1. SUBNET CONFIG
2. Resource Group /Create a New Resource Group
3. VNET
4. Assign Public IP
5. Network Security Group
6. Network Interface Card
7. VM Configuration
8. VM Creation

 
 Example :
The syntax may look a bit odd here. In the -VM parameter, you pass in the $VirtualMachine configuration. But then you return that back to the $VirtualMachine variable. Technically, you are simply replacing whatever had been in the $VirtualMachine variable with the result returned from Set-AzureRmVMOperatingSystem. Conceptually though, you are continuing to build enough information into a variable to create a virtual machine.
You pass in the $VirtulaMachine variable built so far, along with the publisher, offer, SKU, and version. This adds to the VM configuration telling not only that you want Windows, but specifically which operating system and software as well. A note about the $version variable. When you called the New-AzureVM function, you didn’t pass in a version parameter. In the parameter list at the top of the function you’ll see that the $Version parameter defaults to ‘latest’.
 

    # define the RDP username and password of Windows VM to login 
    $UserName='user'
    $Password='user_admin123$'| ConvertTo-SecureString -Force -AsPlainText
    $Credential=New-Object PSCredential($UserName,$Password)
    
    # Create the VM configuration object
    $VmName = "WindowsVirtualMachine"
    $VmSize = "Standard_D2"
    $VirtualMachine = New-AzureRmVMConfig `
      -VMName $VmName `
      -VMSize $VmSize
    # Define Cumputer Name and Credentials 
    $VirtualMachine = Set-AzureRmVMOperatingSystem `
      -VM $VirtualMachine `
      -Windows `
      -ComputerName "WindowsComputer" `
      -Credential $Credential -ProvisionVMAgent
    
    $VirtualMachine = Set-AzureRmVMSourceImage `
      -VM $VirtualMachine `
      -PublisherName "MicrosoftWindowsServer" `
      -Offer "WindowsServer" `
      -Skus "2016-R2Datacenter" `
      -Version "latest"
    
    # Sets the operating system disk properties on a VM.
    $VirtualMachine = Set-AzureRmVMOSDisk `
      -VM $VirtualMachine `
      -CreateOption FromImage | `
      Set-AzureRmVMBootDiagnostics -ResourceGroupName $rgName `
      -StorageAccountName $STA -Enable |`
      Add-AzureRmVMNetworkInterface -Id $NIC.Id
    
    
    # Create the VM.
    New-AzureRmVM `
      -ResourceGroupName $ResourceGroupName `
      -Location $location `
      -VM $VirtualMachine
    

The above Powershell Script is explain indivdually in the follwing manner.

## Step 1 
    Login-AzureRmAccount  

In order to Login into Azure account type the above command and sign in through the azure credentials.
Once login now it will list all subscription which is associated with this account and then select the particular subscription through which we will be creating a Virtual Machine.
For Eg: subscription name “MicrosoftAzure-Subscription-Trail”


    Set-AzureRmContext-SubscriptionName“MicrosoftAzure-Subscription-Trail” 


## Step 2

Now creating a resource group, resource group is where all the services/variables created resides such as Virtual Machine, Azure OS Disk, Network Security Configuration,
etc.
To create a Resource Group (for Eg: PowerShelldemo with location West US)

    $rgName=”PowerShelldemo”
    $location=”West US”
    New-AzureRmResourceGroup-Name $rgName -Location $location


## Step 3

Storage Account - *storage account is required to place VHDs for the virtual machine. The storage account needs to be in the same Azure location as the virtual machine is created, so you can use the same variables for the location and resource group name.*
At this point you now know what you want to create but haven’t yet said where to store it. That’s where the next step comes in. You will need to have a reference to the storage account created (in the previous article) for this purpose.
Type of storage for the managed disk.
If not specified, the disk is created as `Standard_LRS`.
`Standard_LRS` is for Standard HDD.
`StandardSSD_LRS` (added in 2.8) is for Standard SSD.
`Premium_LRS` is for Premium SSD.
`UltraSSD_LRS` (added in 2.8) is for Ultra SSD, which is in preview mode, and only available on select instance types.
See [https://docs.microsoft.com/en-us/azure/virtual-machines/windows/disks-types](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/disks-types) for more information about disk types.

*For Eg : storage account name* *windowsstoragedemo* *with*  *Standard_LRS*

    $STAName= ”windowsstoragedemo”
    $STAType ”Standard_LRS”
    $STA = New-AzureRmStorageAccount -Name $STAName -ResourceGroupName $rgName -Type $STAType -Location $location


## Step 4

*Create a subnet, Virtual Network  (VNet) and NIC Resources, VMs created with the Resource Manage Deployment model require a Resource Manager virtual network, if needed, create a new Resource Manager- based virtual network with at least  one subnet for the new virtual machine.*
For eg : 


    # Create a subnet configuration
    $subnetConfig = New-AzureRmVirtualNetworkSubnetConfig `
      -Name mydemoSubnet `
      -AddressPrefix 192.168.1.0/24
    
    # Create a virtual network
    $vnet = New-AzureRmVirtualNetwork `
      -ResourceGroupName $ResourceGroupName `
      -Location $location `
      -Name MyVnet `
      -AddressPrefix 192.168.0.0/14 `
      -Subnet $subnetConfig
    
    # Create a public IP address and specify a DNS name
    $pip = New-AzureRmPublicIpAddress `
      -ResourceGroupName $ResourceGroupName `
      -Location $location `
      -AllocationMethod Static `
      -IdleTimeoutInMinutes 4 `
      -Name "mypublicdns$(Get-Random)"

**Step 5** 
Create a network interface card to connect VM to a subnet,network security group and public IP address.

    $NIC =New-AzureRmNetworkInterface 
       -Name $NICName 
       -ResourceGroupName $rgName 
       -location $location 
       -SubnetId $vNet.Subnets[0].Id 
       -PublicIpAddressId $pip.Id 
        -PrivateIpAddress “192.168.1.4” 


Step 7 
Connect to VM through by getting public IP address 

    Get-AzureRmPublicIpAddress `
      -ResourceGroupName $ResourceGroupName | Select IpAddress



