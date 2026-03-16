---
description: >-
  Set up SCIM provisioning to automatically manage users and groups in the
  Umbraco Portal from your identity provider.
---

# SCIM Provisioning

SCIM provisioning lets you automate user management in the Umbraco Portal. Instead of manually creating and managing users in the Portal, your identity provider handles it — when a user is added, updated, or deactivated in your identity provider, those changes are automatically pushed to the Portal.

This guide explains what SCIM is, how it works with the Umbraco Portal, and walks you through setting it up with Microsoft Entra ID.

## What is SCIM?

SCIM stands for **System for Cross-domain Identity Management**. It is an open standard (defined in RFC 7643 and RFC 7644) that provides a common way for identity providers and applications to exchange user identity information automatically.

In practical terms, SCIM means your IT team only needs to manage users in one place — your identity provider (such as Microsoft Entra ID). When someone joins the organization, changes roles, or leaves, those changes are automatically reflected in every application that supports SCIM, including the Umbraco Portal.

Without SCIM, administrators need to manually create and deactivate users in the Portal. With SCIM, the identity provider becomes the single source of truth for user identities, and the Portal stays in sync automatically.

## How SCIM Works with the Umbraco Portal

SCIM provisioning follows a push model. Your identity provider acts as the **SCIM client** — it detects changes to users and groups and pushes those changes outward. The Umbraco Portal acts as the **SCIM service provider** — it exposes a set of standard endpoints that receive and apply those changes.

The communication between the two happens over HTTPS using JSON, and is authenticated with a bearer token that is generated for you when you add an identity provider in the Portal.

Here is a simplified view of the flow:

```
Identity Provider (e.g., Entra ID)  →  SCIM API (HTTPS + Bearer Token)  →  Umbraco Portal
```

When the identity provider creates a new user, the Portal receives a SCIM request and creates the corresponding user. When a user is deactivated, the Portal marks them as inactive. The same applies to updates and group membership changes.

{% hint style="info" %}
SCIM provisioning handles the **user lifecycle** (create, update, deactivate, delete) and **group membership**. It does not handle authentication — users still log in via your configured external login provider (for example, OpenID Connect with Entra ID). For more on configuring authentication, see the [External Login Providers](external-login-providers.md) article.
{% endhint %}

## Prerequisites

Before setting up SCIM provisioning, make sure you have the following:

* A **SCIM bearer token**, generated when adding new identity provider in the Umbraco Portal.
* **Admin access** to your identity provider (for example, Microsoft Entra ID).

## Supported SCIM Operations

The Umbraco Portal supports the following SCIM resources and operations:

| Resource | Operations | Description |
|---|---|---|
| **Users** | Create, Read, Update (PUT/PATCH), Delete, List, Filter | Full user lifecycle management |
| **Groups** | Create, Read, Update (PUT/PATCH), Delete, List, Filter | Group membership management |
| **ServiceProviderConfig** | Read | Declares the Portal's SCIM capabilities |
| **ResourceTypes** | Read | Lists supported resource types |
| **Schemas** | Read | Full attribute schema definitions |

{% hint style="info" %}
The Portal supports the `active` attribute for soft-disabling users. When your identity provider deactivates a user, they will no longer be able to log in to the Portal, but their data is preserved.
{% endhint %}

## Setting Up SCIM Provisioning with Microsoft Entra ID

The following walkthrough guides you through configuring SCIM provisioning using Microsoft Entra ID as the identity provider.

### Step 1: Create an Enterprise Application

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com).
2. Navigate to **Identity** > **Applications** > **Enterprise Applications**.
3. Click **New Application**.
4. Click **Create your own application**.
5. Enter a name for the application (for example, "Umbraco Portal SCIM Provisioning").
6. Select **Integrate any other application you don't find in the gallery (Non-gallery)**.
7. Click **Create**.

<!-- placeholder: screenshot of the Enterprise Application creation screen in Entra ID -->

### Step 2: Configure Provisioning

1. In your new Enterprise Application, go to **Provisioning** in the left sidebar.
2. Click **Get started**.
3. Set the **Provisioning Mode** to **Automatic**.

<!-- placeholder: screenshot of the Provisioning Mode set to Automatic -->

### Step 3: Set the Tenant URL and Secret Token

1. In the **Admin Credentials** section:
   * **Tenant URL**: Enter your Portal's SCIM endpoint — `https://portal.umbraco.com/api/scim`
   * **Secret Token**: Paste the SCIM bearer token from your Portal organization settings.
2. Click **Test Connection** to verify that Entra ID can reach your SCIM endpoints.

<!-- placeholder: screenshot of the Admin Credentials section with Tenant URL and Secret Token fields -->

### Step 4: Configure Attribute Mappings

1. Expand the **Mappings** section.
2. Click **Provision Microsoft Entra ID Users**.
3. Review the default attribute mappings. The key mappings are:

| Entra ID Attribute | SCIM Attribute | Notes |
|---|---|---|
| `userPrincipalName` | `userName` | Primary identifier |
| `objectId` | `externalId` | Entra's unique ID for the user |
| `givenName` | `name.givenName` | First name |
| `surname` | `name.familyName` | Last name |
| `mail` | `emails[type eq "work"].value` | Work email |
| `Switch([IsSoftDeleted]...)` | `active` | Controls whether the user is active |
| `displayName` | `displayName` | Display name |

<!-- placeholder: screenshot of the User attribute mappings in Entra ID -->

4. Go back to **Mappings** and click **Provision Microsoft Entra ID Groups**.
5. Review the group mappings — ensure `displayName` and `members` are mapped.

<!-- placeholder: screenshot of the Group attribute mappings in Entra ID -->

{% hint style="info" %}
You can customize which users and groups are provisioned by configuring **Scope** in the provisioning settings. You can either sync all users and groups, or only those assigned to the Enterprise Application.
{% endhint %}

### Step 5: Test the Connection

1. Go back to **Provisioning** > **Overview**.
2. Click **Test Connection** if you have not already done so.
3. After a successful test, click **Provision on demand** to test with a specific user.
4. Select a user and click **Provision** to see a detailed log of the SCIM requests.

<!-- placeholder: screenshot of the Provision on demand screen with a test user selected -->

### Step 6: Start Provisioning

1. Go to **Provisioning** > **Overview**.
2. Set the **Provisioning Status** toggle to **On**.
3. Click **Save**.
4. The initial provisioning cycle will begin. This may take up to 40 minutes depending on the number of users.

<!-- placeholder: screenshot of the Provisioning Status toggle set to On -->

{% hint style="info" %}
Entra ID runs incremental provisioning cycles approximately every 40 minutes. Changes made in Entra ID (new users, group membership changes, deactivations) will be reflected in the Portal after the next cycle. You can trigger an on-demand cycle from the **Provisioning** page.
{% endhint %}

## SCIM Groups

SCIM Groups in the Portal represent group memberships as they exist in your identity provider. When Entra ID pushes group information through SCIM, the Portal stores these as SCIM Group memberships.

It is important to understand that SCIM Groups are **separate from Portal Roles**. Portal Roles control permissions and access within the Portal, while SCIM Groups reflect the group structure in your identity provider.

Currently, SCIM Groups do not directly affect permissions or role assignments in the Portal. In a future release, administrators will be able to configure mappings between SCIM Groups and Portal Roles — for example, mapping a SCIM Group called "Engineers" to the "Developer" role in a specific organization.

{% hint style="info" %}
SCIM Groups represent group memberships as they exist in your identity provider. They are not the same as Portal Roles, which control permissions and access within the Portal.
{% endhint %}

## Troubleshooting

The following table covers common issues you may encounter when setting up or running SCIM provisioning:

| Problem | Cause | Solution |
|---|---|---|
| Test Connection returns "RedactedHtmlContent" | SCIM URL is incorrect or returns HTML instead of JSON | Verify the Tenant URL ends with `/scim` and the endpoint returns JSON |
| Test Connection returns "CredentialValidationUnavailable" | Bearer token is invalid or expired | Regenerate the SCIM token in Portal organization settings |
| Users are created but cannot log in | SCIM handles provisioning, not authentication | Ensure you also have an [external login provider](external-login-providers.md) (for example, OpenID Connect) configured |
| Filter queries return no results | Filter comparison may be case-sensitive | Verify with a known user — filters should be case-insensitive per the SCIM specification |
| Group changes not reflected | Provisioning cycle has not run yet | Wait for the next cycle (~40 minutes) or trigger a manual sync from the Provisioning page |
| PATCH operations fail | Entra ID sends `op` values with capital letters (Add, Replace, Remove) | The Portal handles this — if you see failures, check the provisioning logs for specific errors |

## Related Resources

* [RFC 7643 — SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643)
* [RFC 7644 — SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
* [Microsoft: Tutorial on SCIM provisioning](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups)
* [Microsoft: SCIM Validator](https://scimvalidator.microsoft.com/)
* [External Login Providers](external-login-providers.md) — SCIM handles provisioning (user lifecycle), while external login providers handle authentication (how users log in)
