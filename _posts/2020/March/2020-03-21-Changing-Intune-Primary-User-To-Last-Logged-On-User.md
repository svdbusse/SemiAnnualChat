---
layout: post
date: 2020-03-21
title:  "Bulk change all Intune device Primary Users to last logged on user"
tags: [PowerShell, Microsoft Graph, Intune, Windows 10]
banner: "/SemiAnnualChat/assets/BulkChangePrimaryUser/banner.png"
image: "/assets/BulkChangePrimaryUser/twitter.png"
description: "A PowerShell script to change the Primary User of all Windows 10 Devices in an Intune Tenant to the last logged in user of the device"
author: "Stefan van der Busse"
comments: true
youtubeId:
---
___

With the recent [announcement](https://techcommunity.microsoft.com/t5/intune-customer-success/change-the-intune-primary-user-public-preview-now-available/ba-p/1221264) of the much anticipated ability to change the primary user of devices in Microsoft Intune without the need to reset the device, a number of customers that I work with had the opportunity to go through and update devices to the the correct primary user, and light up new self service Company Portal experiences.

![Example of Microsoft Intune Company Portal](https://docs.microsoft.com/en-us/mem/intune/user-help/media/rs1_appdetailspage_installed_03.png)

As a refresher some of the desirable self service capabilities of the [Company Portal](https://docs.microsoft.com/en-us/mem/intune/user-help/using-the-intune-company-portal-website) of mention for end users are:
* Download and install available applications
* Access bitlocker recovery keys for their device
* Remote wipe or lock their devices

## The Challenge

In the interest of not wanting to spend an entire afternoon wearing out the left click of my mouse, I needed a fast and reliable way to set the primary user to the "current" user of the device for all Windows 10 devices within an Intune tenant.

I was presented with the following challenges:

* Needing up to date information on who is using each device
* An automated or scripted method of changing the primary user, rather than using the Device Management Console


## Finding the current Primary User of every Device

From previous expeditions into the [managedDevice](https://docs.microsoft.com/en-us/graph/api/resources/intune-devices-manageddevice?view=graph-rest-beta) endpoint of the Microsoft Graph, I knew that the last logged on user is stored within the usersLoggedOn property of every Intune Device, in the form of a user objectID and lastLogOnDateTime timestamp.

```json
  "usersLoggedOn": [
    {
      "@odata.type": "microsoft.graph.loggedOnUser",
      "userId": "String",
      "lastLogOnDateTime": "String (timestamp)"
    }
  ]
```
With this information in hand, and confident that 90% of the devices in the environment would belong to the last logged in user I just needed a method of recursively querying and updating each device.

## Bulk changing the Primary User to the Last Logged On User

[Dave Falkus](https://twitter.com/davefalkus) a Program Manager at Microsoft has done all the heavy lifting here, providing [PowerShell script samples](https://github.com/microsoftgraph/powershell-intune-samples/tree/master/ManagedDevices#8-win10_primaryuser_getps1) on GitHub for getting, setting and removing the Primary User for an Intune Device.

Dave's samples are just that - **samples** - to inspire and enable administrators to automate tasks in Microsoft Intune using PowerShell and the Microsoft Graph. The following PowerShell script snippet, in conjunction with Dave's [Win10_PrimaryUser_Set.ps1](https://github.com/microsoftgraph/powershell-intune-samples/blob/master/ManagedDevices/Win10_PrimaryUser_Set.ps1) sample allows us to achieve the following:

1. Get all Windows 10 Devices from the Tenant
2. Foreach device, check who the current Primary User of the Device is
3. Check that the last user that logged in from the usersLoggedOn attribute is not the Primary User of the Device
    - If True: Sets the last logged on user to the Primary User of the device
    - If False: Move onto the next device

```PS
#Get All Windows 10 Intune Managed Devices for the Tenant
$Devices = Get-Win10IntuneManagedDevice

Foreach ($Device in $Devices){

        Write-Host "Device name:" $device."deviceName" -ForegroundColor Cyan
        $IntuneDevicePrimaryUser = Get-IntuneDevicePrimaryUser -deviceId $Device.id

        #Check if there is a Primary user set on the device already
        if($IntuneDevicePrimaryUser -eq $null){

            Write-Host "No Intune Primary User Id set for Intune Managed Device" $Device."deviceName" -f Red

        }

        else {
            $PrimaryAADUser = Get-AADUser -userPrincipalName $IntuneDevicePrimaryUser
            Write-Host "Intune Device Primary User:" $PrimaryAADUser.displayName

        }

        #Get the objectID of the last logged in user for the device, which is the last object in the list of usersLoggedOn
        $LastLoggedInUser = ($Device.usersLoggedOn[-1]).userId

        #Using the objectID, get the user from the Microsoft Graph for logging purposes
        $User = Get-AADUser -userPrincipalName $LastLoggedInUser

            #Check if the current primary user of the device is the same as the last logged in user
            if($IntuneDevicePrimaryUser -notmatch $User.id){

                #If the user does not match, then set the last logged in user as the new Primary User
                $SetIntuneDevicePrimaryUser = Set-IntuneDevicePrimaryUser -IntuneDeviceId $Device.id -userId $User.id

                if($SetIntuneDevicePrimaryUser -eq ""){

                    Write-Host "User"$User.displayName"set as Primary User for device '$($Device.deviceName)'..." -ForegroundColor Green

                }

            }

            else {
                #If the user is the same, then write to host that the primary user is already correct.
                Write-Host "The user '$($User.displayName)' is already the Primary User on the device..." -ForegroundColor Yellow

            }

    Write-Host

}

```

The output of the script will display the following for each device successfully updated:

```
Device name: Laptop-12345
Intune Device Primary User: Megan Bowen
User Joe Bloggs set as Primary User for device 'Laptop-12345'...
```

The full script is available on [Github](https://github.com/svdbusse/IntuneScripts/blob/master/PrimaryUser/Set-PrimaryUserfromLastLogIn.ps1) and comes with no warranty. Please complete your own testing before running against production devices or data.

## Summary

As a number of devices were deployed in the customer's environment using Windows AutoPilot and enrolled using a generic Device Enrollment Manager account to pre-configure devices with apps and device configuration, devices were left with the incorrect Primary User.

*Note: These devices were enrolled before [AutoPilot White Glove](https://docs.microsoft.com/en-us/windows/deployment/windows-autopilot/white-glove) was available, which would now be the recommended solution.*

To correct the Primary User on these devices, this solution allowed us to query the last logged on user of the device (which is not natively available via the Intune UI) and update the Primary User for each device respectively.

Next steps could include:
* Updating the taskbar xml to pin the Company Portal for users to easily find and access
* Assess the applications currently being deployed to devices, and switch some of the portfolio from required to available
* Educate users on being responsible for their own devices, and taking appropriate action if a device is misplaced, or stolen

And don't forget to customise the [Intune Company Portal branding](https://docs.microsoft.com/en-us/mem/intune/apps/company-portal-app), so your users feel at home!