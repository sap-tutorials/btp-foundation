---
parser: v2
auto_validation: true
time: 30
tags: [tutorial>intermediate, technology-platform>sap-btp--kyma-runtime, technology-platform>sap-business-technology-platform]
primary_tag: technology-platform>sap-btp--kyma-runtime
author_name: Nina Hingerl
author_profile: https://github.com/NHingerl
---

# Consume a Shared Service Instance in Kyma

<!-- description -->Create a reference to a shared service instance in a Kyma namespace and consume it, including cross-subaccount scenarios.

## You will learn

- How to create a reference instance in a Kyma namespace using the SAP BTP Operator
- How to create a cross-subaccount reference instance
- How to use selectors instead of hardcoded instance IDs
- How to create a service binding for the reference instance

## Prerequisites

- In [Choose a Service Sharing Method](btp-choose-service-sharing-method), you chose instance sharing for a Kyma consumer application.
- In [Create and Share a Source Service Instance](btp-create-share-source-instance), you created and shared a source instance.
- You have the `id` of the shared source instance from [Create and Share a Source Service Instance](btp-create-share-source-instance). To retrieve it, run `btp get services/instance --subaccount <SUBACCOUNT_ID> --name <INSTANCE_NAME>`.
- Your Kyma cluster has the SAP BTP Operator module enabled.
- You have permissions to create resources in the target namespace.
- For cross-subaccount scenarios: You have the **Subaccount Administrator** role collection in the source subaccount.
- You have installed and configured the [btp CLI](https://help.sap.com/docs/btp/sap-business-technology-platform/download-and-start-using-btp-cli-client).
- You have configured kubectl to connect to your Kyma cluster; see [Access a Kyma Instance Using kubectl](https://help.sap.com/docs/btp/sap-business-technology-platform/3e25944e491049b2aeec68c562a5ee48).
- You have installed [jq](https://jqlang.github.io/jq/download/) for JSON processing in verification steps.

## Intro

### Determine your sharing scope

Is the shared source instance in the same subaccount as your Kyma cluster, or in a different subaccount?

- Same subaccount: Continue with [Create a reference instance](#create-a-reference-instance).
- Different subaccount: Continue with [Create a cross-subaccount reference instance](#create-a-cross-subaccount-reference-instance).

> **Tip:** If your consumer application and the source instance are in different namespaces but the same subaccount, you may not need instance sharing at all. The SAP BTP Operator supports cross-namespace bindings natively. Set `serviceInstanceNamespace` on your `ServiceBinding` to reference a `ServiceInstance` in another namespace directly. For details, see the [ServiceBinding CR](https://kyma-project.io/external-content/btp-manager/docs/user/resources/02-30-service-binding-cr.html).

### Create a reference instance

You create a `ServiceInstance` custom resource that acts as a pointer to the shared source instance. The SAP BTP Operator processes this resource and registers the reference with SAP Service Manager.

1. Create a file named `reference-instance.yaml` with the following content. Replace the placeholder values with your own.

   ```yaml
   apiVersion: services.cloud.sap.com/v1
   kind: ServiceInstance
   metadata:
     name: <REFERENCE_NAME>           # for example, cls-reference
     namespace: <TARGET_NAMESPACE>     # for example, my-app-ns
   spec:
     serviceOfferingName: <SERVICE_NAME>   # for example, cloud-logging
     servicePlanName: reference-instance
     externalName: <REFERENCE_EXTERNAL_NAME>   # display name in SAP Service Manager, for example, cls-ref-from-kyma-prod
     parameters:
       referenced_instance_id: "<SOURCE_INSTANCE_ID>"   # from the previous tutorial's output
   ```

2. To create the reference instance, apply the file:

   ```bash
   kubectl apply -f reference-instance.yaml
   ```

3. To verify that the reference instance is ready, run:

   ```bash
   kubectl get serviceinstance <REFERENCE_NAME> -n <TARGET_NAMESPACE>
   ```

   The **READY** column must show `True`.

   If it shows `False`, check the status conditions for error details:

   ```bash
   kubectl describe serviceinstance <REFERENCE_NAME> -n <TARGET_NAMESPACE>
   ```

When the reference instance is ready, continue with [Use selectors instead of hardcoded IDs (optional)](#use-selectors-instead-of-hardcoded-ids-optional) or [Create a service binding](#create-a-service-binding).

### Create a cross-subaccount reference instance

If your source instance is in the same subaccount, skip this step. You have already created a reference instance in the previous step.

To reference a shared source instance in a different subaccount, you must provide the SAP BTP Operator with access credentials for the source subaccount. You store these credentials in a Kubernetes Secret and reference that Secret from the `btpAccessCredentialsSecret` field in the `ServiceInstance` resource. For an overview, see [Working with Multiple Subaccounts](https://help.sap.com/docs/btp/sap-business-technology-platform/working-with-multiple-subaccounts).

> **Important:**
> - You must set `btpAccessCredentialsSecret` when you create the `ServiceInstance` - you can't add it later.
> - This mechanism only works within the same global account. To share across global accounts, regions, or with external targets, use [Share Service Credentials Manually Across Boundaries](btp-share-credentials-manually) instead.

1. In the source subaccount, make sure an SAP Service Manager instance with the `service-operator-access` plan exists. If Kyma is enabled in the source subaccount, this instance typically exists already.

2. Create a custom Secret in the `kyma-system` namespace of your target cluster with credentials from the source subaccount. Follow the steps in [Creating Your Custom Secret](https://help.sap.com/docs/btp/sap-business-technology-platform/instance-level-mapping?version=Cloud&locale=en-US#creating-your-custom-secret).

   > **Note:** If you create the Secret manually without the `create-secret-file.sh` script, add `tokenurlsuffix: "/oauth/token"` to the Secret's `stringData` section. This value isn't part of the binding output.

3. Create a file named `reference-instance.yaml` with the following content. This is the same pattern as for same-subaccount sharing, with the addition of the `btpAccessCredentialsSecret` field.

   ```yaml
   apiVersion: services.cloud.sap.com/v1
   kind: ServiceInstance
   metadata:
     name: <REFERENCE_NAME>
     namespace: <TARGET_NAMESPACE>
   spec:
     serviceOfferingName: <SERVICE_NAME>
     servicePlanName: reference-instance
     btpAccessCredentialsSecret: btp-access-secret-source-subaccount
     externalName: <REFERENCE_EXTERNAL_NAME>
     parameters:
       referenced_instance_id: "<SOURCE_INSTANCE_ID>"
   ```

   The `btpAccessCredentialsSecret` field tells the Operator to use the specified Secret's credentials (pointing to the source subaccount's SAP Service Manager) when resolving the reference.

4. Apply the file:

   ```bash
   kubectl apply -f reference-instance.yaml
   ```

5. Verify that the reference instance is ready:

    ```bash
    kubectl get serviceinstance <REFERENCE_NAME> -n <TARGET_NAMESPACE>
    ```

    The **READY** column must show `True`.

### Use selectors instead of hardcoded IDs (optional)

Instead of specifying the `referenced_instance_id` directly, you can use selectors to match the shared source instance by name, plan, or label. This approach is useful for automated deployments across different landscapes where instance IDs vary.

The following example shows all available selector types. Use only one selector at a time; comment out or remove the others. Exactly one shared source instance must match the selector. If no instance matches or more than one matches, creation fails.

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: <REFERENCE_NAME>
  namespace: <TARGET_NAMESPACE>
spec:
  serviceOfferingName: <SERVICE_NAME>
  servicePlanName: reference-instance
  parameters:
    selectors:
      # Option 1: Match by instance name (pick one, remove the rest)
      instance_name_selector: "<INSTANCE_NAME>"
      # # Option 2: Match by plan name
      # plan_name_selector: "<PLAN_NAME>"
      # # Option 3: Match by label
      # instance_label_selector:
      #   - "<LABEL_KEY>"
```

### Create a service binding

To get credentials for the shared source instance, create a `ServiceBinding` resource that references the reference instance.

1. Create a file named `reference-binding.yaml`:

   ```yaml
   apiVersion: services.cloud.sap.com/v1
   kind: ServiceBinding
   metadata:
     name: <BINDING_NAME>
     namespace: <TARGET_NAMESPACE>
   spec:
     serviceInstanceName: <REFERENCE_NAME>
   ```

2. Apply the file:

   ```bash
   kubectl apply -f reference-binding.yaml
   ```

3. To verify that the binding is ready and inspect the credentials, run:

   ```bash
   kubectl get servicebinding <BINDING_NAME> -n <TARGET_NAMESPACE>
   ```

The binding creates a Kubernetes Secret with the same name as the binding. This Secret contains the credentials for the original shared source instance. Your application can mount this Secret as a volume or reference it as environment variables.

To inspect the credential keys (without exposing values in a shared terminal), run:

```bash
kubectl get secret <BINDING_NAME> -n <TARGET_NAMESPACE> -o jsonpath='{.data}' | jq 'keys'
```

To verify that credentials aren't empty, decode one field:

```bash
kubectl get secret <BINDING_NAME> -n <TARGET_NAMESPACE> -o jsonpath='{.data.clientid}' | base64 -d
```

### Understand lifecycle behavior

The following table summarizes what happens when you modify shared resources in Kyma.

| Action | Effect |
| --- | --- |
| Delete a reference `ServiceInstance` (`kubectl delete serviceinstance <REFERENCE_NAME>`) | Only the pointer is removed. The source instance and other references aren't affected. For details, see [Deleting Service Bindings and Service Instances](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/5deca69978eb471692743089ce1eed77.html?version=LATEST&locale=en-US). |
| Delete the source instance (using the btp CLI or SAP BTP cockpit) | All reference instances stop working. All bindings become invalid. |
| Unshare the source instance (`btp unshare services/instance`) | No new reference instances can be created. Existing reference instances continue to work. |
| Disable the Kyma environment | If the source instance was created through the btp CLI (as recommended), the source instance isn't affected. If the source instance was created through a `ServiceInstance` resource in the same cluster, it is deprovisioned. |

> **Note:** Creating the source instance through the btp CLI (as done in [Create and Share a Source Service Instance](btp-create-share-source-instance)) rather than through a Kubernetes `ServiceInstance` resource is a deliberate design choice. It protects the instance from accidental deletion if the cluster is recreated. Additionally, Kyma consumers can configure automatic credential rotation using the SAP BTP Operator's `credentialsRotationPolicy` on the `ServiceBinding` resource (opt-in, disabled by default). Cloud Foundry consumers must rotate credentials manually. For details, see [Rotating Service Bindings](https://kyma-project.io/external-content/btp-manager/docs/user/03-40-service-binding-rotation.html).

You have successfully consumed a shared service instance in Kyma. If you want to share other services that don't support instance sharing, see [Share Service Credentials Manually Across Boundaries](btp-share-credentials-manually).

### Test yourself

Question 1: Where must the Kubernetes Secret for cross-subaccount access be created?

- In the same namespace as the reference instance
- In the `default` namespace
- In the `kyma-system` namespace [CORRECT]
- In the source subaccount's Kyma cluster

Question 2: What does `spec.servicePlanName: reference-instance` tell the SAP BTP Operator?

- To create a new source instance with that plan
- To create a lightweight pointer to an existing shared source instance [CORRECT]
- To look for an existing reference plan in the namespace
- To share the instance with all namespaces in the cluster

Question 3: When using selectors in a reference instance instead of a hardcoded `referenced_instance_id`, what happens if multiple shared instances match the selector criteria?

- The most recently created match is used
- The reference instance uses the instance with the lowest resource usage
- The reference instance creation fails [CORRECT]
- All matching instances are combined into one binding

Question 4: What does the SAP BTP Operator create when a `ServiceBinding` for a reference instance becomes ready?

- A new ServiceInstance in the target namespace
- A ConfigMap with the credential fields
- A Kubernetes Secret containing the credentials for the original shared source instance [CORRECT]
- A copy of the shared source instance in the target cluster

Question 5: The `btpAccessCredentialsSecret` mechanism allows cross-subaccount sharing. What is its boundary?

- It works across any two subaccounts worldwide.
- It works only within the same Cloud Foundry organization.
- It works within the same global account (requires SAP Service Manager connectivity). [CORRECT]
- It works only within the same region.
