---
parser: v2
author_name: Madeline Schaefer
author_ profile: https://github.com/Madeline-Schaefer
auto_validation: true
time: 10
tags: [tutorial>beginner, software-product>sap-business-technology-platform, software-product-function>sap-btp-cockpit, tutorial>license, software-product-function>sap-private-link-service, software-product-function>sap-btp-command-line-interface]
primary_tag: software-product-function>sap-private-link-service
---

# Connect SAP Private Link Service to Microsoft Azure Private Link Service (Kyma)
<!-- description --> Connect SAP Private Link service to Microsoft Azure Private Link Service and bind the service instance on a Kyma cluster.

## Prerequisites
- You have a global account and subaccount on SAP Business Technology Platform with SAP Private Link service entitlement: [Set Up SAP Private Link service ](private-link-onboarding).
- You have created a Microsoft Azure Private Link Service in the Azure Portal. You only have to create the Load Balancer resources (pool and rules) and the private link service. The section "Create a private endpoint" can be skipped, as SAP Private Link service will establish the connection for you. See [Create a Private Link service by using the Azure portal](https://docs.microsoft.com/en-us/azure/private-link/create-private-link-service-portal).
When creating the Azure Private Link service, make sure you allowlist the SAP BTP CF Subscription IDs as described in [Only allow requests from SAP BTP CF's Azure subscription](https://help.sap.com/docs/PRIVATE_LINK/42acd88cb4134ba2a7d3e0e62c9fe6cf/844bca7a51f04a15be865b9a6c1867b0.html?locale=en-US&version=CLOUD).
- You have enabled Kyma Runtime and configured kubectl CLI. See [Enable SAP BTP, Kyma Runtime Using the Command Line](https://developers.sap.com/tutorials/btp-cli-setup-kyma-cluster.html)

## You will learn
  - How to create a SAP Private Link service instance to connect to your Microsoft Azure Private Link Service using kubectl CLI
  - How to bind the service instance to your Kyma cluster using kubectl CLI

 SAP Private Link service establishes a private connection between applications running on SAP BTP and selected services in your own IaaS provider accounts. By reusing the private link functionality of our partner IaaS providers, you can access your services through private network connections to avoid data transfer via the public internet.

## Intro
<!-- border -->![Overview of  Link service functionality](private-endpoint.png)

---

### Check offerings of Private Link service


After you've logged in as described in [Enable SAP BTP, Kyma Runtime Using the Command Line](https://developers.sap.com/tutorials/btp-cli-setup-kyma-cluster.html), you can check all available entitlements for your subaccount. Open a command prompt and enter the following command:

```Shell/Bash
btp list accounts/entitlements
```

You can now see a list of service names and service plans, as shown in this example: 

```Shell/Bash
$ btp list accounts/entitlements
Showing entitlements for subaccount 9be57735-1234-1234-1234-0123456789ab:

service name          service plan              quota
...
privatelink           standard                  8
...
```

Make sure you can find `privatelink` under the service name column in the output.


### Get Resource-ID for Azure Private Link Service


To create and enable a private link, you need to define the connection to the service first. To do so, you need the Resource-ID Azure service:

1. Go to the Azure portal.
2. Navigate to the Azure resource for which you want to find out the Resource ID, for example: **Private Link Center** > **Private link services**.
3. Click on **Overview** in the menu on the left side of your screen.

    <!-- border -->![Overview](private-endpoint-Microsoft-azure-overview.png)

4. Click on **JSON View** in the upper right corner of the overview page.
5. Search for the Resource ID in a field at the top of the resulting view in a text box labelled **Resource ID**.

    <!-- border -->![ResourceID](private-endpoint-Microsoft-azure-overview-resource-id.png)

### Make sure SAP BTP Operator module is enabled

SAP BTP Operator module `btp-operator` must be enabled before creating and managing SAP Private Link Service Instances in Kyma. 
Otherwise, you get the following error message: `resource mapping not found for {...} ensure CRDs are installed first`.

Additionally it is required that `btp-operator` module is version `1.1.0` or newer.

To enable the `btp-operator` module, follow the procedure described in [Enable and Disable a Kyma Module.](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module)



### Create private link service


Currently, you do not have any service instances enabled. Therefore, you need to create one. To create a new private link, you need the following information:

- offering (`privatelink`),
- plans (`standard`),
- a unique name (for instance, `privatelink-test`),
- and the Resource-ID from Microsoft Azure (for example, `/subscriptions/<subscription>/resourceGroups/<rg>/providers/Microsoft.Network/privateLinkServices/<my-private-link-service>`).

Enter the following command and fill in the neccessary information:

```Shell/Bash
kubectl create -f - <<EOF
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: privatelink-test
spec:
  serviceOfferingName: privatelink
  servicePlanName: standard
  parameters:
    resourceId: "/subscriptions/<subscription>/resourceGroups/<rg>/providers/Microsoft.Network/privateLinkServices/<privatelink-test>"
    requestMessage: Please Approve
EOF
```

If the creation of the service instance was accepted, you receive a success message telling you to proceed.

> **Tip**: You can edit `requestMessage: ...` to contain any text that provides more information to the approver.

> **Tip**: Depending on the chosen service type, you might need additional parameters. Please check the [list of supported services](https://help.sap.com/docs/private-link/private-link1/consume-azure-services-in-sap-btp) and what are the required parameters in your case.


### Check status of private link


To check the current status of the newly created service instance, you need the name of your service instance (in this example `privatelink-test`). Type in the following:

```Shell/Bash
kubectl get ServiceInstance privatelink-test -o wide
```

Under "status", "ready", and "message", you can see the current status.

```Shell/Bash
NAME              OFFERING     PLAN      STATUS             READY   AGE   ID            MESSAGE
privatelink-test  privatelink  standard  CreateInProgress   False   13s   <some UUID>   ServiceInstance is being created
```

>  Execute this command again, in case there's no change in the current status. If you receive an error message, go back to the previous steps.


> **Security Info**: In a scenario in which the initiator of the private link connection doesn't have access to the Azure Portal to approve the newly private endpoint connection him- or herself, please reach out to the person responsible for approving the connection and share the endpoint name responsibly.


### Approve connection in Azure


Return to Microsoft Azure portal:

1. Select **Settings > Private endpoint connections**.
2. Search for the name of the private endpoint you received from the success message in the previous step.
3. Select the private end point and click **Approve**.

<!-- border -->![Approve your private endpoint](Private-endpoint-approve-connection-azure.png)

You should now receive a success message that the approval is pending.

> **Security Info**: In a scenario in which the person that approves the private endpoint connection wasn't the one that created the Private Link service in the first place, please verify that the connection originated from a trustworthy origin (for instance, a colleague asking for approval via e-mail). This verification process prevents malicious misuse of resource ids. See also [Best Practices for Secure Endpoint Approval](https://help.sap.com/products/PRIVATE_LINK/42acd88cb4134ba2a7d3e0e62c9fe6cf/844bca7a51f04a15be865b9a6c1867b0.html?locale=en-US&version=CLOUD).


### Ensure private link was created successfully


To check the current status of the newly created service instance, you need the name of your service instance (in this example `privatelink-test`). Type in the following:

```Shell/Bash
kubectl get ServiceInstance privatelink-test -o wide
```

You should see the following success message:

```Shell/Bash
NAME              OFFERING     PLAN      STATUS    READY   AGE   ID            MESSAGE
privatelink-test  privatelink  standard  Created   True    2m    <some UUID>   ServiceInstance provisioned successfully
```


### Create a service binding for the service instance


When service binding is created Private Link service enables network access to the IP address associated with the Private Endpoint.

To create a new binding, you need the following information:

- unique name of the binding (for example ```privatelink-binding-test```)
- the name of the service instance (```privatelink-test```)
- unique name of the secret that will be created (for example ```privatelink-secret-test```)

Enter the following command to create a new binding with the example information:

```Shell/Bash
kubectl create -f - <<EOF
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: privatelink-binding-test
spec:
  serviceInstanceName: privatelink-test
  secretName: privatelink-secret-test
EOF
```

If the command is successful, a kubernetes secret with the same name as specified in `secretName` is created. The secret stores information about the private link endpoint in its `hostname` field. Use the following command to print the `hostname` field from the secret:

```Shell/Bash
kubectl get secret privatelink-secret-test -o jsonpath='{.data.hostname}' | base64 --decode
```

As an example the output might look like the following:

```Shell/Bash
{"fqdn":"someresource.privatelink.blob.core.windows.net","ip_addresses":["10.250.1.5"]}
```
Follow the steps in [Configure DNS on Kyma.](https://help.sap.com/docs/private-link/private-link-internal/configure-dns-on-kyma?state=DRAFT)

---

Congratulations! You have successfully completed the tutorial.

---
