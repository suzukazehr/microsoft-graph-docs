---
title: "Tutorial: Use the Privileged Identity Management (PIM) API to assign Azure AD roles"
description: "Learn how to use the Privileged Identity Management (PIM) API in Microsoft Graph to assign Azure AD privileged roles."
author: "FaithOmbongi"
ms.localizationpriority: medium
ms.prod: "governance"
---

# Tutorial: Use the Privileged Identity Management (PIM) API to assign Azure AD roles

Microsoft Graph PIM API enables organizations to manage privileged access to resources in Azure Active Directory (Azure AD). It also helps to manage the risks of privileged access by limiting when access is active, managing the scope of access, and providing an auditable log of privileged access.

In this tutorial, a fictitious company called Contoso Limited wishes to have its IT Helpdesk manage the lifecycle of employees’ access. The company has identified the Azure AD User Administrator role as the appropriate privileged role required by IT Helpdesk, and will use the PIM API to assign the role.

You'll create a role-assignable security group for IT Helpdesk and using the PIM API, assign the security group eligibility to the User Administrator role. By assigning the eligible role to a security group, Contoso has a more efficient way to manage administrator access to resources such as Azure AD roles. For example:

+ Removing existing or adding more group members also removes administrators.
+ Adding more roles to the group members instead of assigning roles to individual users.

Assigning eligibility instead of a persistently active User Administrator privilege allows the company to enforce **just-in-time access**, which grants temporary permissions to carry out the privileged tasks. After defining the role eligibility, the eligible group member then activates their assignment for a temporary period. All records of role activations will be auditable by the company.

>[!NOTE]
>The response objects shown in this tutorial might be shortened for readability.

## Prerequisites

To complete this tutorial, you need the following resources and privileges:

+ A working Azure AD tenant with an Azure AD Premium P2 or EMS E5 license enabled.
+ Sign in to [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer) as a user in a Global Administrator role.
  + [Optional] Start a new incognito or InPrivate browser session, or start a session in an anonymous browser. You'll sign in later in this tutorial.
+ The following delegated permissions: `User.ReadWrite.All`, `Group.ReadWrite.All`, `Directory.Read.All`, `RoleEligibilitySchedule.ReadWrite.Directory`, and `RoleAssignmentSchedule.ReadWrite.Directory`, and `RoleManagement.ReadWrite.Directory`.
+ Authenticator app installed on your phone to register a user for multifactor authentication (MFA).

To consent to the required permissions in Graph Explorer:
1. Select the horizontal ellipses icon to the right of the user account details, and then choose **Select permissions**.
  
      :::image type="content" source="/graph/images/GE-Permissions/selectpermissions.png" alt-text="Select Microsoft Graph permissions." border="true":::

2. Scroll through the list of permissions to these permissions:
    + Group (2), expand and then select **Group.ReadWrite.All**.
    + Directory (4), expand and then select **Directory.Read.All**.
    + RoleAssignmentSchedule (2), expand and then select **RoleAssignmentSchedule.ReadWrite.Directory**.
    + RoleEligibilitySchedule (2), expand and then select **RoleEligibilitySchedule.ReadWrite.Directory**.
    + RoleManagement (3), expand and then select **RoleManagement.ReadWrite.Directory**.
    + User (8), expand and then select **User.ReadWrite.All**.
   
   Select **Consent**, and then select **Accept** to accept the consent of the permissions. For the `RoleEligibilitySchedule.ReadWrite.Directory` and `RoleAssignmentSchedule.ReadWrite.All` permissions, consent on behalf of your organization.

      :::image type="content" source="/graph/images/GE-Permissions/User.ReadWrite.All-consent.png" alt-text="Consent to Microsoft Graph permissions." border="true":::

## Step 1: Create a test user

Create a user who must reset their password at first sign in. From this step, record the value of the new user's **id** for use in the next step. After creating the user, visit the Azure portal and enable multifactor authentication (MFA) for the user. For more information about enabling MFA, see the [See also](#see-also) section.


### Request

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-createUser"
}-->
```http
POST https://graph.microsoft.com/v1.0/users
Content-Type: application/json

{
    "accountEnabled": true,
    "displayName": "Aline Dupuy",
    "mailNickname": "AlineD",
    "userPrincipalName": "AlineD@Contoso.com",
    "passwordProfile": {
        "forceChangePasswordNextSignIn": true,
        "password": "xWwvJ]6NMw+bWH-d"
    }
}
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.user"
} -->
```http
HTTP/1.1 201 Created
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
    "@odata.id": "https://graph.microsoft.com/v2/29a4f813-9274-4e1b-858d-0afa98ae66d4/directoryObjects/7146daa8-1b4b-4a66-b2f7-cf593d03c8d2/Microsoft.DirectoryServices.User",
    "id": "7146daa8-1b4b-4a66-b2f7-cf593d03c8d2",
    "displayName": "Aline Dupuy",
    "userPrincipalName": "AlineD@Contoso.com"
}
```

## Step 2: Create a security group that can be assigned an Azure AD role

Create a group that’s assignable to an Azure AD role. Assign yourself as the group owner and both you and Aline (the user created in Step 1) as members.

### Request: Create a role-assignable group

Replace `1ed8ac56-4827-4733-8f80-86adc2e67db5` with your **id**.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-createSecurityGroup"
}-->
```msgraph-interactive
POST https://graph.microsoft.com/v1.0/groups
Content-type: application/json

{
    "description": "IT Helpdesk to support Contoso employees",
    "displayName": "IT Helpdesk (User)",
    "mailEnabled": false,
    "mailNickname": "userHelpdesk",
    "securityEnabled": true,
    "isAssignableToRole": true,
    "owners@odata.bind": [
        "https://graph.microsoft.com/v1.0/users/1ed8ac56-4827-4733-8f80-86adc2e67db5"
    ]
}
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.group"
} -->
```
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#groups/$entity",
    "@odata.id": "https://graph.microsoft.com/v2/29a4f813-9274-4e1b-858d-0afa98ae66d4/directoryObjects/e77cbb23-0ff2-4e18-819c-690f58269752/Microsoft.DirectoryServices.Group",
    "id": "e77cbb23-0ff2-4e18-819c-690f58269752",
    "description": "IT Helpdesk to support Contoso employees",
    "displayName": "IT Helpdesk (User)",
    "groupTypes": [],
    "isAssignableToRole": true,
    "mailEnabled": false,
    "mailNickname": "userHelpdesk",
    "securityEnabled": true,
    "securityIdentifier": "S-1-12-1-3883711267-1310199794-258579585-1385637464",
    "visibility": "Private",
    "onPremisesProvisioningErrors": []
}
```

### Request

Assign yourself and Aline as the two members to the security group. In the following request, replace:
+ `e77cbb23-0ff2-4e18-819c-690f58269752` in the URL with the value of the group's **id**.
+ `7146daa8-1b4b-4a66-b2f7-cf593d03c8d2` with the value of Aline's **id**.
+ `1ed8ac56-4827-4733-8f80-86adc2e67db5` with the value of your **id**.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-addGroupMembers"
}-->
```msgraph-interactive
PATCH https://graph.microsoft.com/v1.0/groups/e77cbb23-0ff2-4e18-819c-690f58269752
Content-type: application/json

{
    "members@odata.bind": [
        "https://graph.microsoft.com/v1.0/users/1ed8ac56-4827-4733-8f80-86adc2e67db5",
        "https://graph.microsoft.com/v1.0/users/7146daa8-1b4b-4a66-b2f7-cf593d03c8d2"
    ]
}
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.group"
} -->
```
HTTP/1.1 204 No Content
```

## Step 3: Create a unifiedRoleEligibilityScheduleRequest

Now that you have a security group, assign it as eligible for the User Administrator role. In this step:

+ Create a unifiedRoleEligibilityScheduleRequest object that identifies the group **IT Helpdesk (User)** as eligible for the User Administrator role for one year. Azure AD extends this eligible assignment to the group members, that is, you and Aline.
+ Scope the eligible assignment to your entire tenant. This allows the user admin to use their privilege against all users in your tenant, except higher privileged users such as the Global Administrator.

### Request

Replace `e77cbb23-0ff2-4e18-819c-690f58269752` with the value of the **id** of the **IT Helpdesk (User)** security group. This **principalId** identifies the assignee of eligibility to the User Administrator role. The roleDefinitionId `fe930be7-5e62-47db-91af-98c3a49a38b1` is the global template identifier for the User Administrator role in Azure AD.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-unifiedRoleEligibilityScheduleRequest_create"
}-->
```msgraph-interactive
POST https://graph.microsoft.com/beta/roleManagement/directory/roleEligibilityScheduleRequests
Content-type: application/json

{
    "action": "AdminAssign",
    "justification": "Assign User Admin eligibility to IT Helpdesk (User) group",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/",
    "principalId": "e77cbb23-0ff2-4e18-819c-690f58269752",
    "scheduleInfo": {
        "startDateTime": "2021-07-01T00:00:00Z",
        "expiration": {
            "endDateTime": "2022-06-30T00:00:00Z",
            "type": "AfterDateTime"
        }
    }
}
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.unifiedRoleEligibilityScheduleRequests"
} -->
```
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#roleManagement/directory/roleEligibilityScheduleRequests/$entity",
    "id": "64a8bd54-4591-4f6a-9c77-3e9cb1fdd29b",
    "status": "Provisioned",
    "createdDateTime": "2021-09-03T20:45:28.3848182Z",
    "completedDateTime": "2021-09-03T20:45:39.1194292Z",
    "action": "AdminAssign",
    "principalId": "e77cbb23-0ff2-4e18-819c-690f58269752",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/",
    "isValidationOnly": false,
    "targetScheduleId": "64a8bd54-4591-4f6a-9c77-3e9cb1fdd29b",
    "justification": "Assign User Admin eligibility to IT Helpdesk (User) group",
    "createdBy": {
        "user": {
            "id": "1ed8ac56-4827-4733-8f80-86adc2e67db5"
        }
    },
    "scheduleInfo": {
        "startDateTime": "2021-09-03T20:45:39.1194292Z",
        "expiration": {
            "type": "afterDateTime",
            "endDateTime": "2022-06-30T00:00:00Z"
        }
    },
    "ticketInfo": {}
}
```

## Step 4: Confirm the user's current role assignments

While the group members are now eligible for the User Administrator role, they're still unable to use the role. This is because they're yet to activate their eligibility. You can confirm by checking the user's current role assignments.


### Request

In the following request, replace `7146daa8-1b4b-4a66-b2f7-cf593d03c8d2` with the value of Aline's **id**.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-roleAssignments_list"
}-->
```msgraph-interactive
GET https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments?$filter=principalId eq '7146daa8-1b4b-4a66-b2f7-cf593d03c8d2'
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.unifiedRoleAssignments"
} -->
```
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#roleManagement/directory/roleAssignments",
    "value": []
}
```

The empty response object shows that Aline has no existing Azure AD roles in Contoso. Aline will now activate their eligible User Administrator role for a limited time.

## Step 5: User self-activates their eligible assignment

An incident ticket CONTOSO: Security-012345 has been raised in Contoso's incident management system and the company requires that all employee's refresh tokens be invalidated. As a member of IT Helpdesk, Aline is responsible for fulfilling this task.

First, start the Authenticator app on your phone and open Aline Dupuy's account.

Sign in to Graph Explorer as Aline. You may use an incognito session or an anonymous browser for this step. By doing so, you won't interrupt your current session as a user in the Global Administrator role. Alternatively, you can interrupt your current session by signing out of Graph Explorer and signing back in as Aline.

Signed in as Aline, you'll first change your password because this was specified during account creation. Then, because the administrator configured your account for MFA, you'll be prompted to set up your account in the Authenticator app and be challenged for MFA sign-in. This is because PIM requires that MFA for all active role assignments.

After signing in, activate your User Administrator role for five hours.

### Request

To activate a role, call the `roleAssignmentScheduleRequests` endpoint. In this request, the `UserActivate` action allows you to activate your eligible assignment, in this case for five hours.

+ For **principalId**, supply the value of your (Aline's) **id**.
+ The **roleDefinitionId** is the **id** of the role you're eligible for, in this case, the User Administrator role.
+ Enter the details of the ticket system that provides an auditable justification for activating the request.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-roleAssignmentScheduleRequests_selfActivate"
}-->
```msgraph-interactive
POST https://graph.microsoft.com/beta/roleManagement/directory/roleAssignmentScheduleRequests
Content-type: application/json

{
    "action": "SelfActivate",
    "principalId": "7146daa8-1b4b-4a66-b2f7-cf593d03c8d2",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/",
    "justification": "Need to invalidate all app refresh tokens for Contoso users.",
    "scheduleInfo": {
        "startDateTime": "2021-09-04T15:13:00.000Z",
        "expiration": {
            "type": "AfterDuration",
            "duration": "PT5H"
        }
    },
    "ticketInfo": {
        "ticketNumber": "CONTOSO:Security-012345",
        "ticketSystem": "Contoso ICM"
    }
}
```

### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.roleEligibilityScheduleRequests"
} -->

```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#roleManagement/directory/roleAssignmentScheduleRequests/$entity",
    "id": "295edd40-4646-40ca-89b8-ab0b46b6f60e",
    "status": "Granted",
    "createdDateTime": "2021-09-03T21:10:49.6670479Z",
    "completedDateTime": "2021-09-04T15:13:00Z",
    "action": "SelfActivate",
    "principalId": "7146daa8-1b4b-4a66-b2f7-cf593d03c8d2",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/",
    "isValidationOnly": false,
    "targetScheduleId": "295edd40-4646-40ca-89b8-ab0b46b6f60e",
    "justification": "Need to invalidate all app refresh tokens for Contoso users.",
    "createdBy": {
        "user": {
            "id": "7146daa8-1b4b-4a66-b2f7-cf593d03c8d2"
        }
    },
    "scheduleInfo": {
        "startDateTime": "2021-09-04T15:13:00Z",
        "expiration": {
            "type": "afterDuration",
            "endDateTime": null,
            "duration": "PT5H"
        }
    },
    "ticketInfo": {
        "ticketNumber": "CONTOSO:Security-012345",
        "ticketSystem": "Contoso ICM"
    }
}
```

You may confirm your assignment by running `GET https://graph.microsoft.com/beta/roleManagement/directory/roleAssignmentScheduleRequests/filterByCurrentUser(on='principal')`. The response object returns your newly activated role assignment with its status set to `Granted`. With your new privilege, carry out any allowed actions within five hours your assignment is active for. This includes invalidating all employees' refresh tokens. After five hours, the active assignment expires but through your membership in the **IT Support (Users)** group, you still remain eligible for the User Administrator role.

Back in the global administrator session, you have received notifications of both the eligible assignment and the role activation. This allows the global administrator to be aware of all elevations to administrator privileges across your organization.

## Step 6: Clean up resources

Sign in as the Global Administrator and delete the following resources that you created for this tutorial: the role eligibility request, IT Support (Users) group, and the test user (Aline Dupuy).

### Revoke the role eligibility request

#### Request

Replace `e77cbb23-0ff2-4e18-819c-690f58269752` with the **id** of IT Support (Users) group.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-roleEligibilityScheduleRequests_revoke"
}-->
```msgraph-interactive
POST https://graph.microsoft.com/beta/roleManagement/directory/roleEligibilityScheduleRequests
Content-type: application/json

{
    "action": "AdminRemove",
    "principalId": "e77cbb23-0ff2-4e18-819c-690f58269752",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/"
}

```

#### Response

<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.roleEligibilityScheduleRequests"
} -->
```http
HTTP/1.1 201 Created
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#roleManagement/directory/roleEligibilityScheduleRequests/$entity",
    "id": "dcd11a1c-300f-4d17-8c7a-523830400ec8",
    "status": "Revoked",
    "action": "AdminRemove",
    "principalId": "e77cbb23-0ff2-4e18-819c-690f58269752",
    "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
    "directoryScopeId": "/"
}
```

### Delete the IT Support (Users) group

#### Request

Replace `e77cbb23-0ff2-4e18-819c-690f58269752` with the **id** of IT Support (Users) group.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-group_delete"
}-->
```msgraph-interactive
DELETE https://graph.microsoft.com/beta/groups/e77cbb23-0ff2-4e18-819c-690f58269752
```

#### Response

<!-- {
  "blockType": "response",
  "truncated": false
} -->
```http
HTTP/1.1 204 No Content
```

### Delete the test user

#### Request

Replace `7146daa8-1b4b-4a66-b2f7-cf593d03c8d2` with the value of Aline's **id**.

<!-- {
  "blockType": "request",
  "name": "tutorial-assignaadroles-user_delete"
}-->
```msgraph-interactive
DELETE https://graph.microsoft.com/beta/users/7146daa8-1b4b-4a66-b2f7-cf593d03c8d2
```

#### Response

<!-- {
  "blockType": "response",
  "truncated": false
} -->
```http
HTTP/1.1 204 No Content
```

## See also

+ [Start using Privileged Identity Management](/azure/active-directory/privileged-identity-management/pim-getting-started)
+ [Azure AD built-in roles](/azure/active-directory/roles/permissions-reference#all-roles)
+ [Enable per-user Azure AD multifactor authentication to secure sign-in events](/azure/active-directory/authentication/howto-mfa-userstates)
+ [unifiedRoleEligibilityScheduleRequest resource type](/graph/api/resources/unifiedroleeligibilityschedulerequest?view=graph-rest-beta&preserve-view=true)