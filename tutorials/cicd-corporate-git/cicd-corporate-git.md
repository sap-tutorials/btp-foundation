---
parser: v2
author_name: Sarah Lendle
author_profile: https://github.com/SarahLendle
auto_validation: true
time: 30
tags: [ tutorial>beginner, topic>cloud, programming-tool>sapui5, software-product>sap-connectivity-service, software-product>sap-fiori, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-connectivity-service
---

# Connect SAP Continuous Integration and Delivery with Your Corporate Git
<!-- description --> Set up SAP Continuous Integration and Delivery, use SAP Connectivity service to connect it with your corporate Git repository, and create and run a basic CI/CD job.

## Prerequisites
 - You have an account on SAP Business Technology Platform. See [Trial Accounts](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/046f127f2a614438b616ccfc575fdb16.html) or [Enterprise Accounts](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/171511cc425c4e079d0684936486eee6.html).
 - You're an administrator of your global account and Org Manager of your subaccount on SAP Business Technology Platform. See [About Roles in the Cloud Foundry Environment](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/09076385086b4da3bd1808d5ef572862.html).
 - You've installed the Cloud Connector of the SAP Connectivity service. See [SAP BTP Connectivity > Installation](https://help.sap.com/viewer/cca91383641e40ffbe03bdc78f00f681/Cloud/en-US/57ae3d62f63440f7952e57bfcef948d3.html).
 - You have an SAP Fiori project in the Cloud Foundry environment in a public Git repository in your corporate network, which is not directly accessible from the Internet.

## You will learn
  - How to set up SAP Continuous Integration and Delivery
  - How to connect SAP Continuous Integration and Delivery with your corporate Git using SAP Connectivity service
  - How to create and trigger a basic SAP Continuous Integration and Delivery job


### Set up SAP Continuous Integration and Delivery

Enable SAP Continuous Integration and Delivery, add the required permissions, and access the service.

 
1. In your subaccount in the SAP BTP cockpit, choose **Services** → **Service Marketplace**.

2. Search for `Continuous Integration & Delivery` and choose the appearing service tile.

    ![Service Tile](cicd_tile.png)

    > **Note:** If in the **Service Marketplace**, you can't see the **Continuous Integration & Delivery** tile, you might need to add the required entitlements to your subaccount. See [Configure Entitlements and Quotas from Your Global Account](https://help.sap.com/docs/btp/sap-business-technology-platform/configure-entitlements-and-quotas-for-subaccounts#configure-entitlements-and-quotas-from-your-global-account).

3. Choose **Create**.
   
4. In the **New Instance or Subscription** pop-up, select the **Subscription** plan and choose **Create**.

5. From the navigation pane, choose **Security** **&rarr;** **Users**.

    >**Note:** If you use an enterprise account, you need to be a User & Role Administrator of your subaccount to view the **Security** section. See [Managing Subaccounts Using the Cockpit](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/55d0b6d8b96846b8ae93b85194df0944.html).

6. Choose the name of your user.

7. From the **Role Collections** section, choose **...** **&rarr;** **Assign Role Collection**.

8. From the dropdown list, select **CICD Service Administrator** and **CICD Service Developer**. Confirm your choice with **Assign Role Collection**.

    ![Assigning the CI/CD Roles](CICD_roles.png)

9.  Navigate back to your subaccount overview and from the navigation pane, choose **Services** → **Service Marketplace**.

10. Search for `Continuous Integration & Delivery` and choose the appearing service tile.

11. Choose **Go to Application**.

    ![Accessing the CI/CD service](go-to-application.png)

As a result, the user interface of the SAP Continuous Integration and Delivery service opens.


### Set up the connection with the SAP Cloud Connector

Add your SAP BTP subaccount to the SAP Cloud Connector.

1. In the SAP Cloud Connector, choose **Connector** **&rarr;** **Add Subaccount**.

    >**Note:** If you haven't added a subaccount to the Cloud Connector yet, the procedure is slightly different. In this case, you start at **Define Subaccount** and fill in the **First Subaccount** form. The required values, however, are the same.

2. In the **Add Subaccount** pop-up, enter the following information:

    >**Note:** You can find all information about your subaccount in your subaccount overview in the SAP BTP cockpit.

    * **Region:** Enter the region in which your subaccount resides. To find the correct region, compare your API endpoint with [Regions](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/350356d1dc314d3199dca15bd2ab9b0e.html#loiof344a57233d34199b2123b9620d0bb41).
    * **Subaccount:** Enter the ID of your subaccount.
    * **Display Name:** Freely choose a name for your subaccount.    
    * **Subaccount User:** Enter the e-mail address that relates to your subaccount. 
    * **Password:** Enter the password for your subaccount.
    * **Location ID:** Freely choose a location ID for your subaccount. You'll need this ID for the connection with SAP Continuous Integration and Delivery.
    * **Description:** Optionally, enter a description for your subaccount.

    ![Add subaccount pop-up in the SAP Connectivity service](add-subaccount.png)

3. Choose **Save**.


### Create a virtual system and map it to your corporate Git


In the SAP Connectivity service, add a system mapping and a resource.

1. From the navigation pane in the SAP Connectivity service, choose the display name of your subaccount **&rarr;** **Cloud To On-Premise**.

2. To map a new system, choose **+** *(Add)*.

3. In the **Add System Mapping** pop-up, select **Non-SAP System** as **Back-end Type** and choose **Next**.

4. From the drop-down list, select the protocol of your on-premise Git server, and choose **Next**.

5. Enter your internal host and port, and choose **Next**.

6. For **Virtual Host**, enter `my-git`, and for **Virtual Port**, enter `1080`.

	Thereby, you'll create a virtual host that can be reached through `http://my-git:1080`. The virtual host is also needed to configure CI/CD jobs in SAP Continuous Integration and Delivery.

    >**Note:** Even though the virtual host uses HTTP, the connection for the actual data transfer between the cloud connector and SAP Continuous Integration and Delivery is encrypted (HTTPS).

7. Choose **Next**.

8. As **Principal Type**, select **None**, and choose **Next**.

9. For **Host In Request Header**, select **Use Virtual Host**, and choose **Next**.

10. Optionally, enter a description and choose **Next**.

11. In the summary, select **Check Internal Host**, and choose **Finish**.

    ![Add system mapping pop-up in the SAP Connectivity service](add-system-mapping.png)

12. Make sure that your new mapping is selected, and in the **Resources Of my-git:1080** area, choose **+** *(Add)*.

13. In the **Add Resource** pop-up, enter the path to your repository, for example `user/repository` out of `http://my-corporate-git/user/repository`.

14. Select **Active** and **Path And All Sub-Paths**, and choose **Save**.

    ![Add resource pop-up in the SAP Connectivity service](add-resource.png)


### Configure SAP Continuous Integration and Delivery


In SAP Continuous Integration and Delivery, configure credentials for your cloud connector and add your Git repository.

1. In SAP Continuous Integration and Delivery, go to the **Credentials** tab and choose **+** *(Create credentials)*.

2. In the **Create Credentials** pop-up, enter the following values:
   
    * **Name:** Freely choose a unique name for your credential. Only use lowercase letters, numbers, and hyphens, and a maximum of 253 characters.
    * **Description:** Enter a meaningful description for your credential.
    * **Type:** From the drop-down list, choose **Cloud Connector**.
    * **Location ID:** Enter the location ID of your cloud connector. You can find it in the SAP BTP cockpit under **Connectivity** **&rarr;** **Cloud Connectors**.

3. Choose **Create**.

4. Switch to the **Repositories** tab and choose **+** *(Add repository)*.

5. In the **Add Repository** pop-up, enter the following values:
   
    * **Name:** Freely choose a unique name for your repository. We recommend using a name that refers to the actual repository in your source code management system.
    * **Clone URL:** Enter a composition of the virtual host you've created in step 3.6 and the path of the resource you've created in step 3.13, for example `http://my-git:1080/user/repository`. Please make sure that you use the virtual host instead of the real host name and port of your on-premise system.
    * **Credentials:** As your repository isn't private, leave this field empty.
    * **Cloud Connector:** From the drop-down list, choose your Cloud Connector credential.


### Add a webhook in GitHub

Configure a webhook between your GitHub repository and SAP Continuous Integration and Delivery to automate the builds of your job.

A webhook with GitHub allows you to automate SAP Continuous Integration and Delivery builds: Whenever you push changes to your GitHub repository, a webhook push event is sent to the service to trigger a build of the connected job.

The following graphic illustrates this flow:

![CI/CD service flow when using a webhook](webhook-flow.png)

1. In the **WEBHOOK EVENT RECEIVER** section of the **Add Repository** pane, choose **GitHub** as **Type** from the drop-down list.

2. In the **Webhook Credential** drop-down list, choose **Create Credentials**.

3. In the **Create Credentials** pop-up, enter a name for your credential, which is unique in your SAP BTP subaccount, for example **`webhook`**.
   
4. Choose **Generate** next to **Secret** to generate a random token string.
   
    >**Caution:** Note down the secret as you won't be able to see it again.

5. Choose **Create**.
   
6. Choose **Add**.

7. In the **Repositories** tab in SAP Continuous Integration and Delivery, choose your repository.
   
8. In your repository pane, choose **...** → **Webhook Data**.
   
    ![Way to access webhook data](webhook-data.png)

    As a result, the **Webhook Creation** pop-up opens.

9.  In your project in GitHub, go to the **Settings** tab.

10. From the navigation pane, choose **Webhooks**.

11. Choose **Add webhook**.

12. Enter the **Payload URL**, **Content type**, and **Secret** from the **Webhook Creation** pop-up in SAP Continuous Integration and Delivery. For all other settings, leave the default values.

13. Choose **Add webhook**.


### Create and trigger a basic CI/CD job

Configure a basic job for SAP Fiori projects in the Cloud Foundry environment.

1. In SAP Continuous Integration and Delivery, go to the **Jobs** tab and choose **+** *(Create job)*.

2. In the **General Information** section of the **Create Job** pane, enter the following values:

    * **Job Name:** Freely choose a unique name for your job.
    * **Repository:** From the drop-down list, choose your repository.
    * **Branch:** Enter the branch of your repository for which you want to configure your CI/CD job, for example, `main`.
    * **Pipeline:** From the drop-down list, choose **SAP Fiori in the Cloud Foundry environment**.
    * **Version:** If you create a new job, the latest version is selected by default.
    * **State:** To enable your job, choose **ON**.

3.  In the  **Build Retention** section, keep the default values.

4. In the **Stages** section, choose **Job Editor** as **Configuration Mode**.

5. For the **Build** stage, keep the default values.

6. For the **Acceptance** stage, enter the following values for the **Deploy to Cloud Foundry Space** step:

    * **Application Name:** Enter a unique application name.
    * **API Endpoint:** Enter the URL of your SAP BTP, Cloud Foundry API Endpoint. You can find it in the overview of your subaccount in the SAP BTP cockpit, under the **Cloud Foundry Environment:** tab.
    * **Org Name:** Enter the name of your Cloud Foundry organization. You can also find it in the overview of your subaccount.
    * **Space:** Enter the name of the Cloud Foundry space in which you want to test your application.
    * **Credentials:** From the drop-down list, choose the SAP BTP credentials you created.
   
     ![Deploy to CF Space in the UI](CICD_deploystep.png)

7. Switch all other stages off and choose **Create**.
   
    >**Note:** As this tutorial focuses on how to get started with SAP Continuous Integration and Delivery, we've decided to configure only a very basic CI/CD pipeline in it. For how to configure more elaborate ones, see [Supported Pipelines](https://help.sap.com/docs/continuous-integration-and-delivery/sap-continuous-integration-and-delivery/supported-pipelines?language=en-US&version=Cloud).

8. To run your CI/CD pipeline, create and commit a code change in your GitHub project.

    As a result, a build of the connected job is triggered and a new build tile appears in the **Builds** section of your job. If you choose it, the **Build Stages** view opens and you can watch the individual stages of your build run through.

    >**Note:** The pipeline run might take a few minutes.
