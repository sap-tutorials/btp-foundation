---
parser: v2
auto_validation: true
time: 20
tags: [technology-platform>sap-business-technology-platform, technology-platform>sap-btp--cloud-foundry-runtime-and-environment]
primary_tag: technology-platform>sap-business-technology-platform
author_name: Nina Hingerl
author_profile: https://github.com/NHingerl
---

# Consume a Shared Service Instance in Cloud Foundry

<!-- description -->Create a reference to a shared service instance in a Cloud Foundry space and bind an application to it.

## Prerequisites

- In [Choose a Service Sharing Method](btp-choose-service-sharing-method), you chose instance sharing for a Cloud Foundry consumer application.
- In [Create and Share a Source Service Instance](btp-create-share-source-instance), you created and shared a source instance.
- You have the `id` of the shared instance from [Create and Share a Source Service Instance](btp-create-share-source-instance). To retrieve it, run `btp get services/instance --subaccount <SUBACCOUNT_ID> --name <INSTANCE_NAME>`.
- You have the **Space Developer** role in the target Cloud Foundry space.
- You have installed the cf CLI and are logged in to your Cloud Foundry API endpoint; see [Download and Install the Cloud Foundry CLI](https://help.sap.com/docs/btp/sap-business-technology-platform/4ef907afb1254e8286882a2bdef0edf4).

## You will learn

- How to create a reference instance in a Cloud Foundry space
- How to bind an application to a reference instance
- How to use `cf share-service` as a CF-native alternative

### Create a reference instance

In this step, you create a reference instance in your target Cloud Foundry space. This reference instance is a lightweight pointer to the shared source instance. Applications bind to the reference instance and receive credentials for the original.

1. To create the reference instance, run the following command. Replace `<REFERENCE_NAME>` with a name for the reference instance and `<SOURCE_INSTANCE_ID>` with the `id` from [Create and Share a Source Service Instance](btp-create-share-source-instance).

   ```bash
   cf create-service <SERVICE_NAME> reference-instance <REFERENCE_NAME> -c '{"referenced_instance_id": "<SOURCE_INSTANCE_ID>"}'
   ```

   For example, to create a reference to a shared SAP Cloud Logging instance, run:

   ```bash
   cf create-service cloud-logging reference-instance cls-reference -c '{"referenced_instance_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"}'
   ```

2. To verify that the reference instance was created, run:

   ```bash
   cf service <REFERENCE_NAME>
   ```

   The output shows the reference instance with the plan `reference-instance`.

### Bind an application to the reference instance

To give your application access to the shared service, bind it to the reference instance. The binding credentials connect to the original shared instance.

1. To bind an application, run:

   ```bash
   cf bind-service <APP_NAME> <REFERENCE_NAME>
   ```

2. After binding, restage the application so it loads the new credentials:

   ```bash
   cf restage <APP_NAME>
   ```

3. To verify the binding, run:

   ```bash
   cf env <APP_NAME>
   ```

   The `VCAP_SERVICES` section contains the credentials for the shared service instance.

### Use cf share-service (CF-native alternative)

> **Note:** `cf share-service` is a CF-native alternative to the reference instances in the previous steps. It only works within Cloud Foundry - you can't share across runtimes. For limitations and security considerations, see the [Cloud Foundry documentation on sharing service instances](https://docs.cloudfoundry.org/devguide/services/sharing-instances.html).

You can share the services within one Cloud Foundry org, or even across orgs:

[OPTION BEGIN [Same Org]]

To share a service instance with another space in the same org:

1. Run:

   ```bash
   cf share-service <INSTANCE_NAME> -s <TARGET_SPACE>
   ```

2. Switch to the target space and bind the instance to your application:

   ```bash
   cf target -s <TARGET_SPACE>
   cf bind-service <APP_NAME> <INSTANCE_NAME>
   ```

[OPTION END]

[OPTION BEGIN [Different Org]]

Before sharing with a space in a different org (different subaccount), verify the following prerequisites:

- Your service supports cross-org sharing.
- The target subaccount is entitled to the service plan. Otherwise, the plan isn't visible in the target org's marketplace and the sharing fails.
- You have the **Space Developer** role in both the source and target spaces.

1. In the source org, run:

   ```bash
   cf share-service <INSTANCE_NAME> -s <TARGET_SPACE> -o <TARGET_ORG>
   ```

2. Switch to the target org and space and bind the instance to your application:

   ```bash
   cf target -o <TARGET_ORG> -s <TARGET_SPACE>
   cf bind-service <APP_NAME> <INSTANCE_NAME>
   ```

[OPTION END]

### Understand lifecycle behavior

The following table summarizes what happens when you modify shared resources in Cloud Foundry.

| Action | Effect |
|----|----|
| Delete a reference instance (`cf delete-service <REFERENCE_NAME>`) | Only the pointer is removed. The source instance and other references aren't affected. |
| Delete the source instance | All reference instances stop working. All bindings become invalid. |
| Unshare the source instance (`btp unshare services/instance`) | No new reference instances can be created. Existing reference instances continue to work. |

You have successfully consumed a shared service instance in Cloud Foundry. If you want to share other services that don't support instance sharing, see [Share Service Credentials Manually Across Boundaries](btp-share-credentials-manually).

### Test yourself

Question 1: What is the difference between `cf share-service` and creating a reference instance with the `reference-instance` plan?

- They are the same mechanism with different names.
- `cf share-service` is a CF-native feature. The `reference-instance` plan uses SAP Service Manager and works across environments. [CORRECT]
- `cf share-service` works in both Cloud Foundry and Kyma environments.
- `cf share-service` is deprecated.

Question 2: What parameter in `cf create-service` links the reference instance to the shared source?

- `--offering-name`
- `-c '{"referenced_instance_id": "<SOURCE_INSTANCE_ID>"}'` [CORRECT]
- `--plan-name reference-instance`
- `-s <TARGET_SPACE>`

Question 3: After binding a CF app to a reference instance, what must you do for the application to pick up the new credentials?

- Delete and recreate the application
- Run `cf restage <APP_NAME>` [CORRECT]
- Run `cf unbind-service` first, then bind again
- Update the `VCAP_SERVICES` environment variable manually
