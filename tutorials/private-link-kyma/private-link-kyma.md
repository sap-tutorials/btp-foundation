---
parser: v2
author_name: Madeline Schaefer
author_ profile: https://github.com/Madeline-Schaefer
auto_validation: true
time: 10
tags: [tutorial>beginner, software-product-function>sap-private-link-service, products>sap-business-technology-platform, tutorial>license, software-product-function>sap-btp-cockpit, software-product>sap-integration-suite]
primary_tag: software-product>sap-business-technology-platform
---

# Set Up SAP Private Link Service
<!-- description --> Get onboarded to use SAP Private Link service in SAP BTP. 

## Prerequisites
- You have a global account and subaccount on SAP Business Technology Platform. See [Getting Started with SAP Business Technology Platform](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/144e1733d0d64d58a7176e817fa6aeb3.html).

## You will learn
  - How to start with SAP Private Link service
  - How to enable SAP Private Link service in BTP cockpit

## Intro
SAP Private Link service establishes a private connection between applications running on SAP BTP and selected services in your own IaaS provider accounts. By reusing the private link functionality of our partner IaaS providers, you can access your services through private network connections to avoid data transfer via the public internet.

<!-- border -->![Overview of SAP Private Link service functionality](private-endpoint.png)

---

### Set entitlements

To be able to use the functionalities of SAP Private Link service, you first need to set the entitlements in your subaccount. For more information, see [Configure Entitlements and Quotas for Subaccounts](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/5ba357b4fa1e4de4b9fcc4ae771609da.html).

If you have a Feature Set B account, you can use directories to group subaccounts. In order to assign a service to a subaccount that is part of a directory that handles entitlements, you have to assign it to the directory first. For more information, see [Cloud Management Tools - Feature Set Overview](https://help.sap.com/docs/btp/sap-business-technology-platform/cloud-management-tools-feature-set-overview).

1. Navigate to your **global account** of SAP BTP cockpit and select **Account Explorer**.

    <!-- border -->![Access Account Explorer](private-endpoint_Account-Explorer.png)

2. From the list, choose the respective **subaccount** for which you would like to enable SAP Private Link service.
3. Navigate to **Entitlements** in the left hand navigation bar. You can now see all the existing service assignments for this respective subaccount.  
4. Select **Configure Entitlements**.

    <!-- border -->![Configure Entitlements for SAP Private Link service](private-endpoint-configure-entitlements.png)  

5. To add a new service assignment, select **Add Service Plans**. You see now all the offerings available to this specific subaccount.

6. Select **SAP Private Link service**, **standard** (from Available Plans) and then **Add 1 Service Plan**.

    <!-- border -->![Add Service Plan for SAP Private Link service](private-endpoint-add-service-plan.png)



### Define quota in your subaccount


You can distribute entitlements and quotas across subaccounts within a global account. Under **Remaining Global Quota**, you see how much quota is still available across your global account. See also [Manage Entitlements Using the Cockpit](btp-cockpit-entitlements).

Use **+** and **-** to increase or decrease the quota of the SAP Private Link service plan according to your needs.

<!-- border -->![SAP Private Link service quota overview](private-endpoint-quota-overview.png)

> One unit equals one Private Link endpoint.
>
> 

### Enable Kyma runtime


You can enable Kyma runtime by following one of the tutorials:

 - [Enable SAP BTP, Kyma Runtime Using the Command Line](https://developers.sap.com/tutorials/btp-cli-setup-kyma-cluster.html)
 - [Enable SAP BTP, Kyma Runtime](https://developers.sap.com/tutorials/cp-kyma-getting-started.html)


Congratulations! You have successfully completed the onboarding of SAP Private Link service. Continue with [Connect SAP Private Link Service to Microsoft Azure Private Link Service with Kyma](private-link-microsoft-azure).



---