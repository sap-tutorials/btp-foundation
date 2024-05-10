---
parser: v2
auto_validation: true
time: 30
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
author_name: Jamie Cawley
author_ profile: https://github.com/jcawley5
---

# Deploy Commerce Mock Application in SAP BTP, Kyma Runtime
<!-- description --> Deploy and connect the Commerce mock application to SAP BTP, Kyma runtime.

## Prerequisites
  - [Git](https://git-scm.com/downloads) installed
  - [Application Connector module added](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module)

## You will learn
  - How to deploy the Kyma mock application and expose its API to the Internet using an APIRule
  - How to pair the mock application with SAP BTP, Kyma runtime to register the APIs and events from the application

## Intro
The Kyma mock application contains lightweight substitutes for SAP applications to ease the development and testing of extension and integration scenarios based on [`Varkes`](https://github.com/kyma-incubator/varkes). Together with SAP BTP, Kyma runtime, it allows for efficient implementation of application extensions without the need to access the standard SAP applications during development.

---

### Clone the Git repository

1. Go to the [xf-application-mocks](https://github.com/SAP-samples/xf-application-mocks) repository. 
   Within the repo you can find each of the mock applications and their Deployment files within the respective folder. The process outlined in this tutorial is the same for each, but focuses on configuring the commerce mock.

2. Use the green **Code** button to choose one of the options to download the code locally, or simply run the following command using your CLI at your desired folder location:

    ```Shell/Bash
    git clone https://github.com/SAP-samples/xf-application-mocks
    ```

### Apply resources to SAP BTP, Kyma runtime

1. Open Kyma dashboard using the **Console URL** link in SAP BTP cockpit.

2. Go to **Namespaces** and choose **Create**.

3. Provide the name `dev`, toogle the **Enable Sidecar Injection** button, and choose **Create**.

    > Namespaces separate objects inside a Kubernetes cluster. The concept is similar to folders in a file system. Each Kubernetes cluster has a `default` namespace to begin with. Choosing a different value for the namespace will require adjustments to the provided samples.

    > Toggling the **Enable Sidecar Injection** button allows the Istio service mesh to inject the Envoy sidecar proxy into Pods located in this namespace.

4. Open the `dev` Namespace, if it is not already open, and choose **Upload YAML**. 

5. Either copy the contents of the file `/xf-application-mocks/commerce-mock/deployment/k8s.yaml` into the window or use the upload option. Notice that this file contains the resource definitions for the Deployment as well as the Service and the Persistent Volume Claim.

6. Go to **Discovery and Network > API Rules** and choose **Create**.

7. Enter the following values:

    * **Name:** `commerce-mock`
    * **Service Name:** `commerce-mock`
    * **Host:** `commerce`

8. Mark the **GET** and **POST** methods, and choose **Create** to create the APIRule.

    > Even APIRules can be created by describing them within YAML files. You can find the YAML definition of the `APIRule` at `/xf-application-mocks/commerce-mock/deployment/kyma.yaml`.

### Open Commerce mock application

1. Go to **Discovery and Network > API Rules**, and open the mock application in the browser by choosing the **Host** value `https://commerce.*******.kyma.ondemand.com`. If you receive the error `upstream connect...`, the application may have not finished starting. Wait for a minute or two and try again.

2. Leave the mock application open in the browser, you will use it later.

### Create a System

In this step, you will create a System in SAP BTP cockpit that will be used to pair the mock application with SAP BTP, Kyma runtime. You will perfrom this step at the **Global** account level of your SAP BTP account.

1. Open your global SAP BTP account and choose the **System Landscape** menu option.

2. Choose **Add System** under the **Systems** tab.

3. Provide the name `commerce-mock`, set the type to **SAP Commerce Cloud** and then choose **Add**.

4. Choose the option **Get Token**, copy the **Token** value and close the window. This value will expire in five minutes and will be needed in a subsequent step.

    > If the token expires before use, you can obtain a new one by choosing the `Get Token` option shown next to the entry in the Systems list.

### Create a Formation

In this step, you will create a Formation. A Formation is used to connect one or more Systems created in the SAP BTP to a runtime. You will perform this step at the **Global** account level of your SAP BTP account.

1. Within your global SAP BTP account, choose the **System Landscape** menu option. Choose the tab **Formations** and choose the **Create Formation** option.

2. Provide the name `mock-formation`, choose `Side-by-Side Extensibility with Kyma` for the **Formation Type**, and your **Subaccount** where SAP BTP, Kyma runtime is enabled. Choose **Next Step**.

3. Select `commerce-mock` system and choose **Next Step**.
   
4. Choose **Create**.

### Pair an application

The pairing process will establish trust between the Commerce mock application and, in this case, SAP BTP, Kyma runtime. Once the pairing is complete, the registration of APIs and business events can be performed. This process allows developers to utilize the APIs and business events with the authentication aspects handled automatically.

1. Navigate back to the mock application browser window and choose **Connect**. 
2. Paste the copied value in the token text area and then choose **Connect**. If the token has expired, you may receive an error. Simply return to [Step 4](#create-a-system) and generate a new token.

3. Choose **Register All** to register the APIs and events from the mock application.

### Verify setup

1. Navigate back to the Kyma home workspace by choosing **Back to Cluster Details**.

2. In the Kyma home workspace, choose **Integration > Applications**.

3. Choose the **mp-commerce-mock** application by clicking on the name value shown in the list.

You should now see a list of the APIs and events the mock application is exposing.

**Congratulations!** You have successfully configured the Commerce mock application.

---
