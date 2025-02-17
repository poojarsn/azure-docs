---
title: Create a trust relationship between an app and an external identity provider
description: Set up a trust relationship between an app in Azure AD and an external identity provider.  This allows a software workload outside of Azure to access Azure AD protected resources without using secrets or certificates. 
services: active-directory
author: rwike77
manager: CelesteDG

ms.service: active-directory
ms.subservice: develop
ms.topic: how-to
ms.workload: identity
ms.date: 03/30/2022
ms.author: ryanwi
ms.custom: aaddev
ms.reviewer: dastruck, udayh, vakarand
#Customer intent: As an application developer, I want to configure a federated credential on an app registration so I can create a trust relationship with an external identity provider and use workload identity federation to access Azure AD protected resources without managing secrets.
---

# Configure an app to trust an external identity provider (preview)

This article describes how to create a trust relationship between an application in Azure Active Directory (Azure AD) and an external identity provider (IdP).  You can then configure an external software workload to exchange a token from the external IdP for an access token from Microsoft identity platform. The external workload can access Azure AD protected resources without needing to manage secrets (in supported scenarios).  To learn more about the token exchange workflow, read about [workload identity federation](workload-identity-federation.md).  You establish the trust relationship by configuring a federated identity credential on your app registration by using Microsoft Graph or the Azure portal.

Anyone with permissions to create an app registration and add a secret or certificate can add a federated identity credential.  If the **Users can register applications** switch in the [User Settings](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/UserSettings) blade is set to **No**, however, you won't be able to create an app registration or configure the federated identity credential.  Find an admin to configure the federated identity credential on your behalf.  Anyone in the Application Administrator or Application Owner roles can do this.

After you configure your app to trust an external IdP, configure your software workload to get an access token from Microsoft identity provider and access Azure AD protected resources.

## Prerequisites
[Create an app registration](quickstart-register-app.md) in Azure AD.  Grant your app access to the Azure resources targeted by your external software workload.  

Find the object ID of the app (not the application (client) ID), which you need in the following steps.  You can find the object ID of the app in the Azure portal.  Go to the list of [registered applications](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) in the Azure portal and select your app registration.  In **Overview**->**Essentials**, find the **Object ID**.

Get the information for your external IdP and software workload, which you need in the following steps.

The Microsoft Graph beta endpoint (`https://graph.microsoft.com/beta`) exposes REST APIs to create, update, delete [federatedIdentityCredentials](/graph/api/resources/federatedidentitycredential?view=graph-rest-beta&preserve-view=true) on applications. Launch [Azure Cloud Shell](https://portal.azure.com/#cloudshell/) and sign in to your tenant in order to run Microsoft Graph commands from AZ CLI.

## Configure a federated identity credential on an app

When you configure a federated identity credential on an app, there are several important pieces of information to provide.

*issuer* and *subject* are the key pieces of information needed to set up the trust relationship. *issuer* is the URL of the external identity provider and must match the `issuer` claim of the external token being exchanged.  *subject* is the identifier of the external software workload and must match the `sub` (`subject`) claim of the external token being exchanged. *subject* has no fixed format, as each IdP uses their own - sometimes a GUID, sometimes a colon delimited identifier, sometimes arbitrary strings. The combination of `issuer` and `subject` must be unique on the app.  When the external software workload requests Microsoft identity platform to exchange the external token for an access token, the *issuer* and *subject* values of the federated identity credential are checked against the `issuer` and `subject` claims provided in the external token. If that validation check passes, Microsoft identity platform issues an access token to the external software workload.

> [!IMPORTANT]
> If you accidentally add the incorrect external workload information in the *subject* setting the federated identity credential is created successfully without error.  The error does not become apparent until the token exchange fails.

*audiences* lists the audiences that can appear in the external token.  This field is mandatory, and defaults to "api://AzureADTokenExchange". It says what Microsoft identity platform must accept in the `aud` claim in the incoming token.  This value represents Azure AD in your external identity provider and has no fixed value across identity providers - you may need to create a new application registration in your IdP to serve as the audience of this token.

*name* is the unique identifier for the federated identity credential, which has a character limit of 120 characters and must be URL friendly. It is immutable once created.

*description* is the un-validated, user-provided description of the federated identity credential. 

### GitHub Actions example

# [Azure CLI](#tab/azure-cli)

Run the [create a new federated identity credential](/graph/api/application-post-federatedidentitycredentials?view=graph-rest-beta&preserve-view=true) command on your app (specified by the object ID of the app).  Specify the *name*, *issuer*, *subject*, and other parameters.

For examples, see [Configure an app to trust a GitHub repo](workload-identity-federation-create-trust-github.md?tabs=microsoft-graph).

# [Portal](#tab/azure-portal)

Find your app registration in the [App Registrations](https://aka.ms/appregistrations) experience of the Azure portal.  Select **Certificates & secrets** in the left nav pane, select the **Federated credentials** tab, and select **Add credential**.

Select the **GitHub Actions deploying Azure resources** scenario from the dropdown menu.  Fill in the **Organization**, **Repository**,  **Entity type**, and other fields.

For examples, see [Configure an app to trust a GitHub repo](workload-identity-federation-create-trust-github.md?tabs=azure-portal).

---

### Kubernetes example

# [Azure CLI](#tab/azure-cli)

Run the following command to configure a federated identity credential on an app and create a trust relationship with a Kubernetes service account.  Specify the following parameters:

- *issuer* is your service account issuer URL (the [OIDC issuer URL](../../aks/cluster-configuration.md#oidc-issuer-preview) for the managed cluster or the [OIDC Issuer URL](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters/oidc-issuer.html) for a self-managed cluster).  
- *subject* is the subject name in the tokens issued to the service account. Kubernetes uses the following format for subject names: `system:serviceaccount:<SERVICE_ACCOUNT_NAMESPACE>:<SERVICE_ACCOUNT_NAME>`.
- *name* is the name of the federated credential, which cannot be changed later.
- *audiences* lists the audiences that can appear in the 'aud' claim of the external token. This field is mandatory, and defaults to "api://AzureADTokenExchange".

```azurecli
az rest --method POST --uri 'https://graph.microsoft.com/beta/applications/f6475511-fd81-4965-a00e-41e7792b7b9c/federatedIdentityCredentials' --body '{"name":"Kubernetes-federated-credential","issuer":"https://aksoicwesteurope.blob.core.windows.net/9d80a3e1-2a87-46ea-ab16-e629589c541c/","subject":"system:serviceaccount:erp8asle:pod-identity-sa","description":"Kubernetes service account federated credential","audiences":["api://AzureADTokenExchange"]}' 
```

And you get the response:
```azurecli
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#applications('f6475511-fd81-4965-a00e-41e7792b7b9c')/federatedIdentityCredentials/$entity",
  "audiences": [
    "api://AzureADTokenExchange"
  ],
  "description": "Kubernetes service account federated credential",
  "id": "51ecf9c3-35fc-4519-a28a-8c27c6178bca",
  "issuer": "https://aksoicwesteurope.blob.core.windows.net/9d80a3e1-2a87-46ea-ab16-e629589c541c/",
  "name": "Kubernetes-federated-credential",
  "subject": "system:serviceaccount:erp8asle:pod-identity-sa"
}
```

# [Portal](#tab/azure-portal)

Find your app registration in the [App Registrations](https://aka.ms/appregistrations) experience of the Azure portal.  Select **Certificates & secrets** in the left nav pane, select the **Federated credentials** tab, and select **Add credential**.

Select the **Kubernetes accessing Azure resources** scenario from the dropdown menu.

Fill in the **Cluster issuer URL**, **Namespace**, **Service account name**, and **Name** fields:

- **Cluster issuer URL** is the [OIDC issuer URL](../../aks/cluster-configuration.md#oidc-issuer-preview) for the managed cluster or the [OIDC Issuer URL](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters/oidc-issuer.html) for a self-managed cluster.
- **Service account name** is the name of the Kubernetes service account, which provides an identity for processes that run in a Pod. 
- **Namespace** is the service account namespace.
- **Name** is the name of the federated credential, which cannot be changed later.

---

### Other identity providers example

# [Azure CLI](#tab/azure-cli)

Run the following command to configure a federated identity credential on an app and create a trust relationship with an external identity provider.  Specify the following parameters (using a software workload running in Google Cloud as an example):

- *name* is the name of the federated credential, which cannot be changed later.
- *ObjectID*: the object ID of the app (not the application (client) ID) you previously registered in Azure AD.
- *subject*: must match the `sub` claim in the token issued by the external identity provider.  In this example using Google Cloud, *subject* is the Unique ID of the service account you plan to use.
- *issuer*: must match the `iss` claim in the token issued by the external identity provider. A URL that complies with the OIDC Discovery spec. Azure AD uses this issuer URL to fetch the keys that are necessary to validate the token. In the case of Google Cloud, the *issuer* is "https://accounts.google.com".
- *audiences*: must match the `aud` claim in the external token. For security reasons, you should pick a value that is unique for tokens meant for Azure AD. The Microsoft recommended value is "api://AzureADTokenExchange".

```azurecli
az rest --method POST --uri 'https://graph.microsoft.com/beta/applications/<ObjectID>/federatedIdentityCredentials' --body '{"name":"GcpFederation","issuer":"https://accounts.google.com","subject":"112633961854638529490","description":"Testing","audiences":["api://AzureADTokenExchange"]}'
```

And you get the response:
```azurecli
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#applications('f6475511-fd81-4965-a00e-41e7792b7b9c')/federatedIdentityCredentials/$entity",
  "audiences": [
    "api://AzureADTokenExchange"
  ],
  "description": "Testing",
  "id": "51ecf9c3-35fc-4519-a28a-8c27c6178bca",
  "issuer": "https://accounts.google.com"",
  "name": "GcpFederation",
  "subject": "112633961854638529490"
}
```

# [Portal](#tab/azure-portal)

Find your app registration in the [App Registrations](https://aka.ms/appregistrations) experience of the Azure portal.  Select **Certificates & secrets** in the left nav pane, select the **Federated credentials** tab, and select **Add credential**.

Select the **Other issuer** scenario from the dropdown menu.

Specify the following fields (using a software workload running in Google Cloud as an example):

- **Name** is the name of the federated credential, which cannot be changed later.
- **Subject identifier**: must match the `sub` claim in the token issued by the external identity provider.  In this example using Google Cloud, *subject* is the Unique ID of the service account you plan to use.
- **Issuer**: must match the `iss` claim in the token issued by the external identity provider. A URL that complies with the OIDC Discovery spec. Azure AD uses this issuer URL to fetch the keys that are necessary to validate the token. In the case of Google Cloud, the *issuer* is "https://accounts.google.com".

---

## List federated identity credentials on an app

# [Azure CLI](#tab/azure-cli)
Run the following command to [list the federated identity credential(s)](/graph/api/application-list-federatedidentitycredentials?view=graph-rest-beta&preserve-view=true) for an app (specified by the object ID of the app):

```azurecli
az rest -m GET -u 'https://graph.microsoft.com/beta/applications/f6475511-fd81-4965-a00e-41e7792b7b9c/federatedIdentityCredentials' 
```

And you get a response similar to:

```azurecli
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#applications('f6475511-fd81-4965-a00e-41e7792b7b9c')/federatedIdentityCredentials",
  "value": [
    {
      "audiences": [
        "api://AzureADTokenExchange"
      ],
      "description": "Testing",
      "id": "1aa3e6a7-464c-4cd2-88d3-90db98132755",
      "issuer": "https://token.actions.githubusercontent.com/",
      "name": "Testing",
      "subject": "repo:octo-org/octo-repo:environment:Production"
    }
  ]
}
```

# [Portal](#tab/azure-portal)

Find your app registration in the [App Registrations](https://aka.ms/appregistrations) experience of the Azure portal.  Select **Certificates & secrets** in the left nav pane and select the **Federated credentials** tab.  The federated credentials that are configured on your app are listed.

---

## Delete a federated identity credential

# [Azure CLI](#tab/azure-cli)

Run the following command to [delete a federated identity credential](/graph/api/application-list-federatedidentitycredentials?view=graph-rest-beta&preserve-view=true) from an app (specified by the object ID of the app):

```azurecli
az rest -m DELETE  -u 'https://graph.microsoft.com/beta/applications/f6475511-fd81-4965-a00e-41e7792b7b9c/federatedIdentityCredentials/51ecf9c3-35fc-4519-a28a-8c27c6178bca' 

```

# [Portal](#tab/azure-portal)

Find your app registration in the [App Registrations](https://aka.ms/appregistrations) experience of the Azure portal.  Select **Certificates & secrets** in the left nav pane and select the **Federated credentials** tab.  The federated credentials that are configured on your app are listed.

To delete a federated identity credential, select the **Delete** icon for the credential.

---

## Next steps
- To learn how to use workload identity federation for Kubernetes, see [Azure AD Workload Identity for Kubernetes](https://azure.github.io/azure-workload-identity/docs/quick-start.html) open source project. 
- To learn how to use workload identity federation for GitHub Actions, see [Configure a GitHub Actions workflow to get an access token](/azure/developer/github/connect-from-azure).
- For more information, read about how Azure AD uses the [OAuth 2.0 client credentials grant](v2-oauth2-client-creds-grant-flow.md#third-case-access-token-request-with-a-federated-credential) and a client assertion issued by another IdP to get a token.
- For information about the required format of JWTs created by external identity providers, read about the [assertion format](active-directory-certificate-credentials.md#assertion-format).