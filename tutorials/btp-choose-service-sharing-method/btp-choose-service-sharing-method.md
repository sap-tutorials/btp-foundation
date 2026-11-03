---
parser: v2
auto_validation: true
time: 15
tags: [tutorial>intermediate, technology-platform>sap-business-technology-platform]
primary_tag: technology-platform>sap-business-technology-platform
author_name: Nina Hingerl
author_profile: https://github.com/NHingerl
---

# Choose a Service Sharing Method

<!-- description -->Learn about the different methods for sharing service instances across runtimes and subaccounts and determine which method fits your scenario.

## You will learn

- Which methods exist for sharing services across runtimes and subaccounts
- How to determine which method fits your scenario
- When a service doesn't need sharing at all

## Prerequisites

- You have a service instance that you want to make available in a different Cloud Foundry space, Kyma namespace, or subaccount.
- You have access to the SAP BTP cockpit with the **Subaccount Viewer** role or higher.

## Intro

### Understand why you share service instances

Applications often run in different scopes: separate Cloud Foundry spaces, Kyma namespaces, or even different subaccounts. In many cases, these applications need to access the same backend service, for example, a shared logging service, a credential store, or a database.

In every sharing scenario, there are two roles:

- **Producer:** The scope that owns the service instance and provides credentials.
- **Consumer:** The scope where an application uses those credentials.

The following tutorials use operational terms based on these roles. The **source instance** is the original service instance you share. The **target** is the space, namespace, or subaccount that receives access. The **consumer application** is the app that uses the shared credentials.

Without a sharing mechanism, you would need to create a separate service instance in each target scope. This increases cost, complicates management, and fragments data.

> **Important:** Some services must not be shared because they are security-sensitive or scoped to a specific space or namespace. Check the service's documentation before sharing. For a list of such services, see [Sharing SAP BTP Services](https://help.sap.com/docs/BTP/df50977d8bfa4c9a8a063ddb37113c43/9fe7e160bf58488aa42e726ebe3c7111.html?locale=en-US&version=LATEST).

### Learn the sharing methods

> **Note:** This tutorial introduces the group [Share BTP Services Across Runtimes and Subaccounts](https://developers.sap.com/group.share-btp-services-across-runtimes.html). The following tutorials cover instance sharing and manual credential sharing. Additional methods exist, such as cross-namespace bindings in Kyma or service-specific methods. For a full comparison, see [Sharing SAP BTP Services](https://help.sap.com/docs/BTP/df50977d8bfa4c9a8a063ddb37113c43/9fe7e160bf58488aa42e726ebe3c7111.html?locale=en-US&version=LATEST).

- **Centralized service:** Some services are designed to be globally accessible. You create an instance centrally and access it through its URL, API, or destinations.
  
- **Instance sharing** (also called platform-managed sharing): This is the recommended method and uses the SAP Service Manager. You create one source instance (the producer), mark it as "shared," and create lightweight reference instances in target scopes. Bindings on the reference instance yield credentials to the original. The platform securely manages the lifecycle and credentials.

- **Manual credential sharing**: This is a universal fallback that works across any boundary, including different subaccounts, regions, and global accounts. You create a service key for the source instance, extract the credentials, and inject them into another scope as a User-Provided Service (Cloud Foundry) or Kubernetes Secret (Kyma). Because credentials live outside the platform's management, you are responsible for their transport, rotation, and cleanup.

The following steps help you determine which sharing method applies to your scenario.

### Check if your service is centralized

If you're not sure whether your service is architecturally centralized, check the service's documentation, or see [Centralized Services](https://help.sap.com/docs/btp/btp-admin-guide/sharing-btp-services?locale=en-US&version=Cloud#centralized-services).

- If your service is centralized (for example, SAP Cloud Transport Management), create an instance (or one per landscape stage). You don't have to set up any sharing mechanism and this tutorial group doesn't apply. If you still want to learn how sharing works, pick a different service and continue to the next step.

- If your service isn't centralized, continue to the next step.

### Check for runtime-native options

If you use Cloud Foundry, continue to the next step. This tutorial group covers the CF-native `cf share-service` command.

If you use Kyma: Are your source and target in the same Kyma cluster, just in different namespaces?

- Yes: Use a cross-namespace `ServiceBinding`. Set `serviceInstanceNamespace` on your `ServiceBinding` to reference the service instance's namespace directly. No sharing mechanism is needed and this tutorial group doesn't apply. For details, see [ServiceBinding CR](https://kyma-project.io/external-content/btp-manager/docs/user/resources/02-30-service-binding-cr.html).
- No: Continue to the next step.

### Check if the service supports instance sharing

- Yes, using the standard `reference-instance` plan: Continue to the next step.
- Yes, but the service uses a service-specific method: Follow the service's sharing documentation; this tutorial group doesn't apply.
- No: Use manual credential sharing. Continue with [Share Service Credentials Manually Across Boundaries](btp-share-credentials-manually).

> **Tip**: If you're not sure, verify it in your subaccount: open **Instances and Subscriptions**, find the instance, and look for a **Share** option in the actions menu. Alternatively, check the service's documentation for `supportsInstanceSharing: true`.

### Choose your target runtime

If your service supports instance sharing, start with [Create and Share a Source Service Instance](btp-create-share-source-instance).

Then, continue with the tutorial for your target runtime:

- Cloud Foundry: [Consume a Shared Service Instance in Cloud Foundry](btp-consume-shared-instance-cf)
- Kyma: [Consume a Shared Service Instance in Kyma](btp-consume-shared-instance-kyma)

Both tutorials cover same-subaccount and cross-subaccount scenarios.

> **Note:** If you want to share across regions or global accounts, instance sharing alone isn't sufficient. Use manual credential sharing (covered in [Share Service Credentials Manually Across Boundaries](btp-share-credentials-manually)) for the cross-region portion.

### Test yourself

Question 1: Which sharing method integrates with the platform's lifecycle management and authorization model?

- Manual credential sharing
- Instance sharing (SAP Service Manager) [CORRECT]
- Centralized service
- User-Provided Service

Question 2: Your service doesn't support instance sharing. You need to make it available in a Cloud Foundry space in a different subaccount. Which method do you use?

- Instance sharing with reference instances
- `cf share-service` with the `-o` flag
- Manual credential sharing [CORRECT]
- Cross-namespace `ServiceBinding`

Question 3: Your Kyma application and the service instance are in different namespaces of the same cluster, within the same subaccount. What do you do?

- Create a reference instance using instance sharing
- Use manual credential sharing with a Kubernetes Secret
- Set `serviceInstanceNamespace` on a cross-namespace `ServiceBinding` [CORRECT]
- Share the instance using `cf share-service`

Question 4: A service is listed as security-sensitive in its documentation. What do you do?

- Share it using manual credential sharing instead of instance sharing
- Create a reference instance with restricted permissions
- Do not share it. Use a separate instance in each scope. [CORRECT]
- Share it only within the same subaccount
