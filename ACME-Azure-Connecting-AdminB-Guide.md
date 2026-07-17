# ACME Azure Connecting – Step-by-Step Guide for Admin B

> **Terminology note:** Throughout this guide the term **connecting** is used in place of **onboarding**. The underlying scripts and Azure resource names still use the word *onboarding* as a technical identifier – those names must be entered exactly as shown and must not be changed.

> **Context:** This guide covers the manual Azure Portal steps that correspond to the CyberArk connecting automation scripts. Admin B must have the **Global Administrator** role in Entra ID and be **Owner** of the Azure root management group.
>
> The process connects the ACME Entra tenant and all Azure resources to the CyberArk Identity Security Platform (ISP). It registers two CyberArk services:
> - **Connect Cloud Environments (CCE)** – lets CyberArk discover and manage your Azure environment.
> - **Secure Cloud Access (SCA)** – lets CyberArk manage privileged access to Azure resources and Entra.

---

## Reference values

| Item | Value |
|---|---|
| ACME Entra Directory ID | `013b1119-5a71-40ea-8e77-d31c36d5ac5d` |
| CyberArk connecting URL | `https://przd-test3-cloudonboarding.cyberark.cloud` |
| CyberArk Deployment ID | `fdf7d427-f364-46dc-80c6-11b3f12e3953` |
| Root Management Group scope | `/providers/Microsoft.Management/managementGroups/013b1119-5a71-40ea-8e77-d31c36d5ac5d` |

---

## Part 1 – Connect Cloud Environments (CCE) App Registration

### Step 1.1 – Create the CCE App Registration

1. In the Azure Portal, navigate to **Microsoft Entra ID → App registrations → New registration**.
2. Fill in:
   - **Name:** `cyberark-cloud-onboarding-app-entra-tenant-87574a2011764e34bb44`
   - **Supported account types:** Accounts in this organizational directory only (Single tenant)
   - **Redirect URI:** Platform = **Web**, URL = `https://przd-test3-cloudonboarding.cyberark.cloud`
3. Click **Register**. Note the **Application (client) ID** that is displayed – this is `<CCE_APP_ID>`.

### Step 1.2 – Add API permission to the CCE app

1. In the app, go to **API permissions → Add a permission → Microsoft Graph → Application permissions**.
2. Search for and add: **CrossTenantInformation.ReadBasic.All** (ID `cac88765-0581-4025-9725-5ebc13f729ee`).
3. Click **Add permissions**.
4. Click **Grant admin consent for ACME** and confirm. The status column should show a green tick.

### Step 1.3 – Create a Service Principal for the CCE app

A service principal is created automatically when you register an app in your own tenant. Verify it exists:

1. Navigate to **Enterprise applications**, search for `cyberark-cloud-onboarding-app-entra-tenant-87574a2011764e34bb44`.
2. If it does not appear, go back to the App Registration → **Overview** and click **Create a service principal** (or use the "Managed application in local directory" link).

### Step 1.4 – Add a Federated Credential to the CCE app

1. In the App Registration, go to **Certificates & secrets → Federated credentials → Add credential**.
2. Choose **Federated credential scenario:** Other issuer.
3. Fill in:
   - **Issuer:** `https://aci4148.id.cyberark.cloud/Oauth2_For_Azure_Workload_Identity_Federation/`
   - **Subject identifier:** `3f2192e6-9f40-4ecf-ac62-ecfb8e5c3581`
   - **Audience:** `api://AzureADTokenExchange`
   - **Name:** `cloud-onboarding`
4. Click **Add**.

### Step 1.5 – Assign the "Management Group Reader" role to the CCE app

1. In the Azure Portal, navigate to **Management groups**.
2. Select the root management group (the one with ID `013b1119-5a71-40ea-8e77-d31c36d5ac5d`, or named after your ACME tenant).
3. Click **Access control (IAM) → Add role assignment**.
4. Select role: **Management Group Reader**.
5. Assign access to: **User, group, or service principal**.
6. Search for and select `cyberark-cloud-onboarding-app-entra-tenant-87574a2011764e34bb44`.
7. Click **Review + assign**.

---

## Part 2 – Secure Cloud Access (SCA) – Entra-level App Registration

SCA creates **two** app registrations: one for Entra-level operations and one for resource-level operations. Both are scoped to the root management group.

### Step 2.1 – Create the SCA Entra custom role

1. Navigate to **Management groups → Root management group → Access control (IAM) → Add custom role**.  
   *(Alternatively: In Subscription IAM → Custom roles → Add, using the root management group as assignable scope.)*
2. Use **Start from scratch** and set:
   - **Name:** `cyberark-sca-role-entra-e24ffa6e97ff-d31c36d5ac5d`
   - **Description:** `SCA Custom role for Azure Entra | sca-version:0.0.4 | deployment-id:fdf7d427-f364-46dc-80c6-11b3f12e3953`
   - **Baseline permissions:** No permissions (Actions, NotActions, DataActions, NotDataActions all empty)
   - **Assignable scopes:** `/providers/Microsoft.Management/managementGroups/013b1119-5a71-40ea-8e77-d31c36d5ac5d`
3. Save the role.

### Step 2.2 – Create the SCA Entra App Registration

1. Navigate to **Microsoft Entra ID → App registrations → New registration**.
2. Fill in:
   - **Name:** `cyberark-sca-app-entra-e24ffa6e97ff-d31c36d5ac5d`
   - **Supported account types:** Accounts in this organizational directory only (Single tenant)
   - **Redirect URI:** Platform = **Web**, URL = `https://przd-test3-cloudonboarding.cyberark.cloud`
3. Click **Register**. Note the **Application (client) ID** – this is `<SCA_ENTRA_APP_ID>`.

### Step 2.3 – Add API permissions to the SCA Entra app

1. In the app, go to **API permissions → Add a permission → Microsoft Graph → Application permissions**.
2. Add all three of the following permissions:
   - **RoleManagement.Read.Directory** (ID `9e3f62cf-ca93-4989-b6ce-bf83c28f9fe8`)
   - **Group.ReadWrite.All** (ID `62a82d76-70ea-41e2-9197-370581804d09`)
   - **User.ReadBasic.All** (ID `97235f07-e226-4f63-ace3-39588e11d3a1`)
3. Click **Add permissions**.
4. Click **Grant admin consent for ACME** and confirm.

### Step 2.4 – Create a Service Principal for the SCA Entra app

Verify or create the Enterprise Application entry (same as Step 1.3, but for `cyberark-sca-app-entra-e24ffa6e97ff-d31c36d5ac5d`).

### Step 2.5 – Add a Federated Credential to the SCA Entra app

1. In the App Registration, go to **Certificates & secrets → Federated credentials → Add credential**.
2. Choose **Federated credential scenario:** Other issuer.
3. Fill in:
   - **Issuer:** `https://aci4148.id.cyberark.cloud/SCA_Oauth2_For_Azure_Workload_Identity_Federation/`
   - **Subject identifier:** `SCA_ISOLATED_SYSTEM_USER_FOR_AZURE_E24FFA6E97FF_D31C36D5AC5D_ENTRA`
   - **Audience:** `api://AzureADTokenExchange`
   - **Name:** `cyberark-sca-app-entra-user-e24ffa6e97ff-d31c36d5ac5d`
4. Click **Add**.

### Step 2.6 – Assign the SCA Entra custom role to the SCA Entra app

1. Navigate to **Management groups → Root management group → Access control (IAM) → Add role assignment**.
2. Select role: `cyberark-sca-role-entra-e24ffa6e97ff-d31c36d5ac5d`.
3. Assign to service principal: `cyberark-sca-app-entra-e24ffa6e97ff-d31c36d5ac5d`.
4. Click **Review + assign**.

---

## Part 3 – Secure Cloud Access (SCA) – Resource-level App Registration

### Step 3.1 – Create the SCA Resource custom role

1. Navigate to **Management groups → Root management group → Access control (IAM) → Add custom role**.
2. Use **Start from scratch** and set:
   - **Name:** `cyberark-sca-role-resource-e24ffa6e97ff-d31c36d5ac5d`
   - **Description:** `SCA Custom role for Azure Resources | sca-version:0.0.4 | deployment-id:fdf7d427-f364-46dc-80c6-11b3f12e3953`
   - **Assignable scopes:** `/providers/Microsoft.Management/managementGroups/013b1119-5a71-40ea-8e77-d31c36d5ac5d`
3. Under **Permissions → Add**, add the following **Actions** (no DataActions needed):

   | Action |
   |---|
   | `Microsoft.Authorization/roleAssignments/read` |
   | `Microsoft.Authorization/roleAssignments/write` |
   | `Microsoft.Authorization/roleAssignments/delete` |
   | `Microsoft.Authorization/roleDefinitions/read` |
   | `Microsoft.Authorization/roleManagementPolicies/read` |
   | `Microsoft.Authorization/permissions/read` |
   | `Microsoft.ResourceGraph/operations/read` |
   | `Microsoft.ResourceGraph/resources/read` |
   | `Microsoft.Resources/subscriptions/read` |
   | `Microsoft.Resources/subscriptions/resourceGroups/read` |
   | `Microsoft.Resources/subscriptions/resourcegroups/resources/read` |
   | `Microsoft.Resources/subscriptions/providers/read` |
   | `Microsoft.Resources/subscriptions/resources/read` |
   | `Microsoft.Resources/subscriptions/operationresults/read` |
   | `Microsoft.Resources/resources/read` |
   | `Microsoft.Management/managementGroups/read` |
   | `Microsoft.Management/managementGroups/subscriptions/read` |

4. Save the role.

### Step 3.2 – Create the SCA Resource App Registration

1. Navigate to **Microsoft Entra ID → App registrations → New registration**.
2. Fill in:
   - **Name:** `cyberark-sca-app-resource-e24ffa6e97ff-d31c36d5ac5d`
   - **Supported account types:** Accounts in this organizational directory only (Single tenant)
   - **Redirect URI:** Platform = **Web**, URL = `https://przd-test3-cloudonboarding.cyberark.cloud`
3. Click **Register**. Note the **Application (client) ID** – this is `<SCA_RESOURCE_APP_ID>`.

### Step 3.3 – Add API permissions to the SCA Resource app

1. In the app, go to **API permissions → Add a permission → Microsoft Graph → Application permissions**.
2. Add all four of the following permissions:
   - **Group.ReadWrite.All** (ID `62a82d76-70ea-41e2-9197-370581804d09`)
   - **User.ReadBasic.All** (ID `97235f07-e226-4f63-ace3-39588e11d3a1`)
   - **GroupMember.ReadWrite.All** (ID `dbaae8cf-10b5-4b86-a4a1-f871c94c6695`)
   - **Group.Create** (ID `bf7b1a76-6e77-406b-b258-bf5c7720e98f`)
3. Click **Add permissions**.
4. Click **Grant admin consent for ACME** and confirm.

### Step 3.4 – Create a Service Principal for the SCA Resource app

Verify or create the Enterprise Application entry for `cyberark-sca-app-resource-e24ffa6e97ff-d31c36d5ac5d`.

### Step 3.5 – Add a Federated Credential to the SCA Resource app

1. In the App Registration, go to **Certificates & secrets → Federated credentials → Add credential**.
2. Choose **Federated credential scenario:** Other issuer.
3. Fill in:
   - **Issuer:** `https://aci4148.id.cyberark.cloud/SCA_Oauth2_For_Azure_Workload_Identity_Federation/`
   - **Subject identifier:** `SCA_ISOLATED_SYSTEM_USER_FOR_AZURE_E24FFA6E97FF_D31C36D5AC5D_RESOURCE`
   - **Audience:** `api://AzureADTokenExchange`
   - **Name:** `cyberark-sca-app-resource-user-e24ffa6e97ff-d31c36d5ac5d`
4. Click **Add**.

### Step 3.6 – Assign the SCA Resource custom role to the SCA Resource app

1. Navigate to **Management groups → Root management group → Access control (IAM) → Add role assignment**.
2. Select role: `cyberark-sca-role-resource-e24ffa6e97ff-d31c36d5ac5d`.
3. Assign to service principal: `cyberark-sca-app-resource-e24ffa6e97ff-d31c36d5ac5d`.
4. Click **Review + assign**.

---

## Part 4 – Construct the output value for Admin A

After completing Parts 1–3, collect the three App IDs noted earlier and construct the following JSON. Replace the placeholders with the actual IDs.

```json
{
  "entra_id": "013b1119-5a71-40ea-8e77-d31c36d5ac5d",
  "deployment_id": "fdf7d427-f364-46dc-80c6-11b3f12e3953",
  "services": [
    {
      "service_name": "cloud_onboarding",
      "output": "[{\"application_id\":\"<CCE_APP_ID>\",\"is_consent_required\":\"false\"}]"
    },
    {
      "service_name": "sca",
      "output": "[{\"application_id\":\"<SCA_ENTRA_APP_ID>\",\"is_consent_required\":\"false\",\"identity_trusted_user\":\"SCA_ISOLATED_SYSTEM_USER_FOR_AZURE_E24FFA6E97FF_D31C36D5AC5D_ENTRA\",\"add_permissions_to_manage_cluster\":false},{\"application_id\":\"<SCA_RESOURCE_APP_ID>\",\"is_consent_required\":\"false\",\"identity_trusted_user\":\"SCA_ISOLATED_SYSTEM_USER_FOR_AZURE_E24FFA6E97FF_D31C36D5AC5D_RESOURCE\",\"add_permissions_to_manage_cluster\":false}]"
    }
  ]
}
```

Once the JSON is complete (with real App IDs substituted), Base64-encode it using Azure Cloud Shell or any terminal:

```bash
echo -n '<paste the JSON here on one line>' | base64 | tr -d '\n'
```

Provide the resulting Base64 string to **Admin A**, who will paste it into the CyberArk ISP connecting field.

---

## Summary of resources created

| Resource | Name | Scope |
|---|---|---|
| App Registration (CCE) | `cyberark-cloud-onboarding-app-entra-tenant-87574a2011764e34bb44` | ACME Entra tenant |
| App Registration (SCA Entra) | `cyberark-sca-app-entra-e24ffa6e97ff-d31c36d5ac5d` | ACME Entra tenant |
| App Registration (SCA Resource) | `cyberark-sca-app-resource-e24ffa6e97ff-d31c36d5ac5d` | ACME Entra tenant |
| Custom Role (SCA Entra) | `cyberark-sca-role-entra-e24ffa6e97ff-d31c36d5ac5d` | Root management group |
| Custom Role (SCA Resource) | `cyberark-sca-role-resource-e24ffa6e97ff-d31c36d5ac5d` | Root management group |
| Role Assignment (CCE) | Management Group Reader → CCE app | Root management group |
| Role Assignment (SCA Entra) | SCA Entra custom role → SCA Entra app | Root management group |
| Role Assignment (SCA Resource) | SCA Resource custom role → SCA Resource app | Root management group |
