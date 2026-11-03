---
parser: v2
auto_validation: true
time: 15
tags: [technology-platform>sap-business-technology-platform]
primary_tag: technology-platform>sap-business-technology-platform
author_name: Nina Hingerl
author_profile: https://github.com/NHingerl
---

# Create and Share a Source Service Instance

<!-- description -->Create a service instance at the subaccount level and mark it as shared so that other scopes can reference it.

## Prerequisites

- In [Choose a Service Sharing Method](btp-choose-service-sharing-method), you chose instance sharing.
- The service you want to share is entitled in your subaccount.
- You have the **Subaccount Administrator** role collection.
- You have installed and configured the [btp CLI](https://help.sap.com/docs/btp/sap-business-technology-platform/download-and-start-using-btp-cli-client).

## You will learn

- How to create a service instance at the subaccount level using the btp CLI
- How to mark a service instance as shared
- How to verify that the reference-instance plan is available

### Choose a service for this tutorial

This tutorial is service-agnostic and uses SAP Cloud Logging as example. You can use any service whose plan supports standard instance sharing; for a list of supported services and which use service-specific methods instead, see [Sharing SAP BTP Services](https://help.sap.com/docs/BTP/df50977d8bfa4c9a8a063ddb37113c43/9fe7e160bf58488aa42e726ebe3c7111.html?locale=en-US&version=LATEST).

Throughout this tutorial, the commands use placeholder values. Replace them with the values for your chosen service:

| Placeholder | Description | Example (SAP Cloud Logging) |
|---|---|---|
| `<SERVICE_NAME>` | The technical name of the service offering | cloud-logging |
| `<PLAN_NAME>` | The technical name of the service plan | standard |
| `<INSTANCE_NAME>` | A name you choose for your instance | my-shared-cls |
| `<SUBACCOUNT_ID>` | The ID of your subaccount (go to your subaccount's Overview page and find it under General Information) | a1b2c3d4-... |

To find the technical names of services and plans available in your subaccount, run:

```bash
btp list services/offering --subaccount <SUBACCOUNT_ID>
btp list services/plan --subaccount <SUBACCOUNT_ID>
```

### Create the service instance

Create the source service instance at the subaccount level with the btp CLI (rather than a runtime-specific tool). This way, you decouple the instance from the runtime's lifecycle, so that disabling the runtime on that subaccount doesn't affect the service instance.

1. To create the instance, run:

   ```bash
   btp create services/instance --subaccount <SUBACCOUNT_ID> --offering-name <SERVICE_NAME> --plan-name <PLAN_NAME> --name <INSTANCE_NAME>
   ```

2. To verify that the instance was created, run:

   ```bash
   btp get services/instance --subaccount <SUBACCOUNT_ID> --name <INSTANCE_NAME>
   ```

   The output shows the instance details, including its `id`. You need this value later to establish the reference pointer. You can retrieve it at any time by with this command.

> **Tip:** Some services need configuration parameters at creation time. If your service needs parameters, add `--parameters <JSON_OR_FILE>` to the create command. Check the service's documentation for required parameters.

### Share the instance

To make the instance available for reference from other scopes, mark it as shared. 

Run:

```bash
btp share services/instance --subaccount <SUBACCOUNT_ID> --name <INSTANCE_NAME>
```

This command marks the instance as shared in SAP Service Manager. Also, it makes a new service plan named `reference-instance` available for the service. When you consume the shared service, you use this plan to create pointers to the shared instance.

### Verify that sharing is active

1. To confirm that the instance is shared, run:

   ```bash
   btp get services/instance --subaccount <SUBACCOUNT_ID> --name <INSTANCE_NAME>
   ```

   In the output, verify that the `shared` property is `true`.

2. To confirm that the `reference-instance` plan is available, run:

   ```bash
   btp list services/plan --subaccount <SUBACCOUNT_ID> --offering-name <SERVICE_NAME>
   ```

   The output must include a plan named `reference-instance`. If it doesn't, verify that the instance is shared and that the service plan supports sharing.

You now have a shared service instance that other scopes can reference. Depending on your target runtime, continue with:
- [Consume a Shared Service Instance in Cloud Foundry](btp-consume-shared-instance-cf)
-  [Consume a Shared Service Instance in Kyma](btp-consume-shared-instance-kyma).

### Test yourself

Question 1: Why is it recommended to create the shared instance using the btp CLI rather than a runtime-specific tool?

- The btp CLI is faster.
- It decouples the instance from any single runtime's lifecycle. [CORRECT]
- Runtime-specific tools don't support instance creation.
- The btp CLI automatically shares the instance.

Question 2: What happens when you share a service instance?

- A copy of the instance is created in every space.
- The instance credentials are sent to all subaccount administrators.
- A `reference-instance` plan becomes available for the service. [CORRECT]
- The instance is moved to the global account level.

Question 3: How do you confirm that the `reference-instance` plan is available after sharing?

- Run `btp list services/plan` and check that `reference-instance` appears in the output [CORRECT]
- Run `btp get services/instance` and check that `shared` is `true`
- Run `btp share services/instance` again to refresh the plan list
- Check the SAP BTP cockpit under Service Instances
