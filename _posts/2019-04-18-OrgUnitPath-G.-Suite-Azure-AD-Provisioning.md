---
layout: post
date: 2019-04-18
title:  "Provisioning Azure AD users into G. Suite Organization Units"
tags: [Azure AD, G. Suite]
thumbnail: "/SemiAnnualChat/assets/OrgUnitPath/banner.png"
---
___

## Migrating to Azure AD
Recently during a cloud migration project, I supported a customer with the goal of removing all on-premises infrastructure dependencies. Including **Active Directory**.

As a result, the use of [Google Cloud Directory Sync](https://tools.google.com/dlpage/dirsync/) was no longer available to provision users from their replacement directory **Azure AD** through to G. Suite.

Easy, I thought. I'll use Azure AD's user provisioning capabilities to provision users directly from Azure AD. A great cloud to cloud integration, with no middleware dependencies.

Spinning up a demo tenant, within a couple of hours I had provisioning up and running, but I needed to find a method of provisioning users into the correct Organization Unit in G. Suite, rather than into the root of the OU structure.

To achieve this, I'd need to configure an Attribute mapping that sets the "orgUnitPath" user attribute in G. Suite as the account is created/updated.

## Prerequisites:
* Azure AD Premium Plan 1 or 2
* G. Suite (Education or Business) tenant
* User provisioning from Azure AD to G. Suite configured - for details follow this tutorial on [docs.microsoft.com](https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/google-apps-provisioning-tutorial).
* Understanding of the [G. Suite Admin SDK](https://developers.google.com/admin-sdk/directory/v1/guides/manage-users)

**Note**: You will need to configure a second G. Suite Enterprise Application to configure provisioning in, this is a [known issue](https://github.com/MicrosoftDocs/azure-docs/issues/7763#issuecomment-398932730).

## Configuring Attribute Mappings

Initially, the orgUnitPath attribute wasn't available as a target attribute - but after a support call with the Azure AD team, it was thankfully added into the schema:

![Edit Attributes](/SemiAnnualChat/assets/OrgUnitPath/EditAttribute2.png "Edit Attributes")

## Dynamically Provisioning Users to different OUs
As Azure AD does not have an OU or hierarchical structure, the question was: How do I configure dynamic mappings to apply to groups of users to specify their unique OrgUnitPath - rather than just provisioning all users into a single OU by hard coding the Constant Value in the mapping.

I had two options:
1. Use a direct mapping from an Azure AD user attribute to the G. Suite OrgUnitPath attribute
2. Write an expression to utilise existing group membership to specify the destination orgUnitPath

After reviewing how to [Write expressions for attribute mappings](https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/functions-for-customizing-application-data) I identified the **SingleAppRoleAssignment** function:

![Single App Role Assignment Definition](/SemiAnnualChat/assets/OrgUnitPath/SingleAppRoleAssignment.png "Single App Role Assignment Definition")

This function would allow me to create additional Azure AD App Registration Roles, that I could assign to groups of users to specify the desired orgUnitPath.

### Expression Attribute Mapping
The following shows the final expression based attribute mapping used to map App Roles to orgUnitPaths.

#### Adding an Attribute
![Edit Attributes](/SemiAnnualChat/assets/OrgUnitPath/EditAttribute3.png "Edit Attributes")
#### All Attribute Mappings
![Attribute Mappings](/SemiAnnualChat/assets/OrgUnitPath/AttributeMapping.png "Attribute Mappings")

## Adding Azure AD Application Manifest Roles

I won't go into detail here on how to create App Roles in an App Manifest - take a look at this [docs.microsoft.com](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps) article for details.

In each role that you create in the JSON manifest you will need to change the following:

JSON Attribute | Value
------------ | -------------
displayName | The orgUnitPath e.g. /Admin/Accounts
id | A unique GUID ([Generator found here](https://www.guidgenerator.com/online-guid-generator.aspx))
description | A description of the orgUnitPath
value | The orgUnitPath e.g. /Admin/Accounts

**Example 1:**
Setting the path for a top level Org Unit:

```json
"appRoles": [
    {
      "allowedMemberTypes": [
        "User"
      ],
      "displayName": "/Staff",
      "id": "c9543c69-b39a-4cb2-ad4e-db6cd5d73f70",
      "isEnabled": true,
      "description": "G. Suite Staff OrgUnit",
      "value": "/Staff"
    }
  ],
"availableToOtherTenants": false
```

**Example 2:**
Setting the path for a multi level Org Unit:

```json
"appRoles": [
    {
      "allowedMemberTypes": [
        "User"
      ],
      "displayName": "/Staff/Administration/Accounts",
      "id": "bc5e28c1-ab2c-47af-98de-ea4a0fec8dbf",
      "isEnabled": true,
      "description": "G. Suite Accounts Team OrgUnit.",
      "value": "/Staff/Administration/Accounts"
    }
  ],
"availableToOtherTenants": false
```

**Note:** Your value and displayname cannot contain spaces.

#### Mapping Groups to Roles
Now that i'd created my roles with their destinations OUs hard coded into the "value", I could assign my groups of users the application, specifying the corresponding role as part of the assignment.
![Add User Assignments](/SemiAnnualChat/assets/OrgUnitPath/AddAssignment.png "Add User Assignments")

**List of Assigned Roles:**
![Users and Groups](/SemiAnnualChat/assets/OrgUnitPath/UsersAndGroups.png "Users and Groups")

### Audit Logs
As an example, in the audit log we can see a test user Jessica Jones being provisioned into the orgUnitPath "/Students", ready for G. Suite policy and configuration to be applied.
![Audit Logs](/SemiAnnualChat/assets/OrgUnitPath/AuditLog.png "Audit Logs")

### A word of Warning
This is a great solution for getting users provisioned into G. Suite and  into their _initially_ correct OU. But through rather extensive testing and going back and forth with Azure AD support, there's a major caveat that they don't mention in any of the support articles.

All provisioning requests that are sent to G. Suite use a **POST** REST method. This is generally fine; and works great for user creation. But any subsequent changes such as a surname name change, or a user moving between departments, business units which results in a change to the orgUnitPath in G. Suite is sent, but simply not honoured.

The POST method is used by design, so it looks like we are stuck with a one time provisioning action only, and any subsequent user changes will need to be handled manually by your ops team.

2 steps forward, 1 step back. Hopefully this shortcoming is addressed in the near future.