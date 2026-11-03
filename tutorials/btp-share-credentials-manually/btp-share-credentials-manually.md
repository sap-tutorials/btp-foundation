---
parser: v2
auto_validation: true
time: 25
tags: [tutorial>intermediate, technology-platform>sap-business-technology-platform, technology-platform>sap-btp--cloud-foundry-runtime-and-environment, technology-platform>sap-btp--kyma-runtime]
primary_tag: technology-platform>sap-business-technology-platform
author_name: Nina Hingerl
author_profile: https://github.com/NHingerl
---

# Share Service Credentials Manually

<!-- description -->Share service credentials across subaccounts, regions, or global accounts by creating a service key and injecting the credentials into Cloud Foundry or Kyma.

## You will learn

- How to create a service key and extract credentials
- How to inject credentials into Cloud Foundry as a User-Provided Service
- How to inject credentials into Kyma as a Kubernetes Secret
- How to rotate and clean up manually shared credentials

## Prerequisites

- In [Choose a Service Sharing Method](btp-choose-service-sharing-method), you chose manual credential sharing.
- You have the **Subaccount Administrator** role collection in the source subaccount.
- You have installed and configured the btp CLI; see [Download and Start Using the btp CLI Client](https://help.sap.com/docs/btp/sap-business-technology-platform/8a8f17f5fd334fb583438edbd831d506).
- For Cloud Foundry: You have installed the cf CLI and have the **Space Developer** role in the target space; see [Download and Install the Cloud Foundry CLI](https://help.sap.com/docs/btp/sap-business-technology-platform/4ef907afb1254e8286882a2bdef0edf4).
- For Kyma: You have configured kubectl for the target cluster and have permissions to create Secrets in the target namespace; see [Access a Kyma Instance Using kubectl](https://help.sap.com/docs/btp/sap-business-technology-platform/3e25944e491049b2aeec68c562a5ee48).
- For Kyma: You have installed [jq](https://jqlang.github.io/jq/download/) for JSON processing in verification steps.

## Intro

### Understand when to use manual credential sharing

Use manual credential sharing when platform-native instance sharing doesn't cover your scenario. The following table lists common examples:

| Scenario | Reason for manual sharing | Approach |
|----|----|----|
| Cloud Foundry space in a different subaccount | CF doesn't support cross-subaccount reference instances | User-Provided Service |
| Cloud Foundry space in a different region | Reference instances are scoped to a single subaccount | User-Provided Service |
| Kyma namespace in a different region or global account | `btpAccessCredentialsSecret` requires SAP Service Manager connectivity within the same global account | Kubernetes Secret |
| External target (external application, on-premise system) | No platform runtime available | Direct credential injection |

> **Important:** With manual credential sharing, you bypass the platform's lifecycle management. You are responsible for credential rotation, transport security, and cleanup.

### Create a service key on the source instance

A service key generates a set of credentials for the source instance without binding it to a specific application.

1. To create a service key, run:

   ```bash
   btp create services/binding --subaccount <SOURCE_SUBACCOUNT_ID> --instance-name <INSTANCE_NAME> --name <KEY_NAME> --binding-type key
   ```

2. To retrieve the credentials, run:

   ```bash
   btp get services/binding --subaccount <SOURCE_SUBACCOUNT_ID> --name <KEY_NAME>
   ```

3. Copy the credentials JSON from the output; you need it in the next steps.

> **Tip:** Name service keys to indicate their target location (for example, `cls-key-for-eu20-prod-space` if you're sharing an SAP Cloud Logging instance). This naming convention helps you track which key is used where and simplifies rotation.

### Inject credentials into the target runtime

Choose the injection method that matches your target runtime:

[OPTION BEGIN [Cloud Foundry]]

To make the shared credentials available in a Cloud Foundry space, create a [User-Provided Service Instance](https://help.sap.com/docs/btp/sap-business-technology-platform/creating-user-provided-service-instances) (UPSI).

1. To create the UPSI, run the following command. Replace `<CREDENTIALS_JSON>` with the JSON block you copied earlier.

   ```bash
   cf cups <UPS_NAME> -p '<CREDENTIALS_JSON>'
   ```

   The `-p` flag accepts inline JSON. If the JSON is complex, run `cf cups <UPS_NAME> -p` without a value. The CLI prompts you for each key-value pair.

2. To bind your application to the UPSI, run:

   ```bash
   cf bind-service <APP_NAME> <UPS_NAME>
   cf restage <APP_NAME>
   ```

3. To verify the binding, run:

   ```bash
   cf env <APP_NAME>
   ```

   The `VCAP_SERVICES` section contains the credentials under the User-Provided Service entry.

[OPTION END]

[OPTION BEGIN [Kyma]]

To make the shared credentials available in a Kyma namespace, create a Kubernetes Secret.

1. Create a file named `manual-credentials-secret.yaml` with the following content. Replace the placeholder values with the actual credential values you copied earlier.

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: <SECRET_NAME>
     namespace: <TARGET_NAMESPACE>
   stringData:
     clientid: "<CLIENT_ID>"
     clientsecret: "<CLIENT_SECRET>"
     url: "<SERVICE_URL>"
     # Add other fields from the service key as needed
   ```

   > **Tip:** Map each top-level field from the service key JSON to a separate key in `stringData`. This approach lets your application reference individual credential fields as environment variables or volume mounts, rather than parsing a single JSON blob.

2. To create the Secret, run:

   ```bash
   kubectl apply -f manual-credentials-secret.yaml
   ```

3. To verify that the Secret exists and inspect its keys (without exposing values), run:

   ```bash
   kubectl get secret <SECRET_NAME> -n <TARGET_NAMESPACE> -o jsonpath='{.data}' | jq 'keys'
   ```

Your application can now mount this Secret as a volume or reference it in environment variable definitions.

[OPTION END]

For external targets (external applications, on-premise systems, or third-party platforms), inject the credentials using whatever mechanism the target system supports: environment variables, configuration files, vault integration, or direct API configuration. The service key JSON structure is the same regardless of the target.

### Rotate credentials

Because the platform doesn't manage manually shared credentials, you must rotate them according to your organization's security policy. For service-specific rotation recommendations, see [Security Recommendations](https://help.sap.com/docs/btp/sap-btp-security-recommendations-c8a9bb59fe624f0981efa0eff2497d7d/sap-btp-security-recommendations).

1. Create a new service key on the source instance:

   ```bash
   btp create services/binding --subaccount <SOURCE_SUBACCOUNT_ID> --instance-name <INSTANCE_NAME> --name <NEW_KEY_NAME> --binding-type key
   ```

2. Retrieve the new credentials:

   ```bash
   btp get services/binding --subaccount <SOURCE_SUBACCOUNT_ID> --name <NEW_KEY_NAME>
   ```

3. Update the target:

   - For Cloud Foundry: Update the User-Provided Service with the new credentials.

     ```bash
     cf uups <UPS_NAME> -p '<NEW_CREDENTIALS_JSON>'
     cf restage <APP_NAME>
     ```

   - For Kyma: Update the Kubernetes Secret. The following command generates the Secret manifest locally without sending it to the cluster (`--dry-run=client`), then pipes it to `kubectl apply` to update the existing Secret in place.

     ```bash
     # Update the Secret in place (generates YAML locally, then applies it)
     kubectl create secret generic <SECRET_NAME> \
       --namespace <TARGET_NAMESPACE> \
       --from-literal=clientid="<NEW_CLIENT_ID>" \
       --from-literal=clientsecret="<NEW_CLIENT_SECRET>" \
       --from-literal=url="<NEW_SERVICE_URL>" \
       --dry-run=client -o yaml | kubectl apply -f -
     ```

     > **Tip:** If your application reads credentials at startup (not dynamically), restart the application pods after updating the Secret: `kubectl rollout restart deployment/<DEPLOYMENT_NAME> -n <TARGET_NAMESPACE>`. Alternatively, Kyma consumers can configure automatic credential rotation using the SAP BTP Operator's `credentialsRotationPolicy`. For details, see [Rotating Service Bindings](https://kyma-project.io/external-content/btp-manager/docs/user/03-40-service-binding-rotation.html).

4. Delete the old service key:

   ```bash
   btp delete services/binding --subaccount <SOURCE_SUBACCOUNT_ID> --name <OLD_KEY_NAME> --confirm
   ```

### Understand lifecycle behavior

The following table summarizes what happens when you modify manually shared resources.

| Action | Effect |
|----|----|
| Delete the UPSI (`cf delete-service <UPS_NAME>`) or Secret (`kubectl delete secret <SECRET_NAME>`) | Only the local credential copy is removed. The source instance and service key aren't affected. |
| Delete the service key (`btp delete services/binding`) | The credentials become invalid. Applications using the UPSI or Secret fail at the next authentication attempt. |
| Delete the source instance | The credentials become invalid. Applications fail. |
| Rotate the service key without updating the UPSI or Secret | Applications continue to use the old credentials until you delete the old key. |

> **Important:** Use one service key per target location. This limits the blast radius if a key is compromised and simplifies rotation. You can rotate one target at a time without affecting others.

You have successfully shared service credentials manually. For an overview of all sharing methods, see [Choose a Service Sharing Method](btp-choose-service-sharing-method).

### Test yourself

Question 1: You need to share a service instance from a subaccount in the EU10 region with a Cloud Foundry application in the US20 region. Which method do you use?

- Platform-native instance sharing with a reference instance
- `cf share-service`
- Manual credential sharing with a User-Provided Service [CORRECT]
- Cross-subaccount sharing with `btpAccessCredentialsSecret`

Question 2: After rotating a service key, what must you do before deleting the old key?

- Unshare the source instance.
- Update the User-Provided Service or Kubernetes Secret with the new credentials. [CORRECT]
- Restart SAP Service Manager.
- Nothing. The old key is automatically invalidated.

Question 3: Why is it recommended to use one service key per target location?

- The platform limits the number of bindings per key to one.
- Service keys are region-specific.
- It limits the blast radius if a key is compromised and simplifies per-target rotation. [CORRECT]
- The btp CLI doesn't support multiple bindings per key.
