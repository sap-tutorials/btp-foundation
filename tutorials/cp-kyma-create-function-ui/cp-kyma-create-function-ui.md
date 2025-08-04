---
parser: v2
auto_validation: true
time: 15
primary_tag: software-product>sap-btp\, kyma-runtime
tags: [ tutorial>beginner, software-product>sap-business-technology-platform]
author_name: Gaurav Abbi, Oliver Stiefbold, Malgorzata Swieca, Jacek Konopelski
keywords: kyma
---

# Create Kyma Function and a Microservice via UI

<!-- description -->In this tutorial, you will use Kyma UI to create a Kyma Function and a microservice.

## You will learn

  - How to enter Kyma dashboard
  - How to create a Kyma Function
  - How to create a Kyma microservice

## Prerequisites

- You have created and set up your "SAP BTP, Kyma Environment" either manually or by Quick Account Setup.
- You have [enabled Kyma in your subaccount](https://developers.sap.com/tutorials/cp-kyma-getting-started.html).

### Add the Serverless Module to your Kyma cluster

1. Open Kyma dashboard. Go to the **Kyma Environment** section, and choose **Link to dashboard**.
   
2. Select **Modify Modules**, and choose **Add**.
   
3. Check **serverless**, and choose **Add**.

### Create a "Hello-World" Kyma Function in Kyma Dashboard

1. In your Kyma dashboard, go to **Namespaces** and choose the **default** namespace.

2. To enable istio sidecar injection for the **default** namespace, choose **edit** and switch the sidecar injection toggle.
   
    If you create a Function in a namespace with Istio sidecar injection enabled, an Istio sidecar proxy is automatically injected to the Function's Pod during its creation. This makes the Function part of the Istio service mesh. To expose a workload using an APIRule custom resource, it is required to include the workload in the Istio service mesh.

2. Go to **Workloads > Functions** and choose **Create**.

3. Fill in the form in the **Create Function** view using the following details and choose **Create**.

    - **Template**: `Default`
    - **Name**: `hello-world`
    - **Language**: `JavaScript`
    - **Runtime**: `node.js 20`
    - **Function Profile**: `XS`
  
4. Creating a Function takes a few seconds. Select the newly created **hello-world** Function. Scroll down, and notice that it does not have any APIRules yet, you need to create one to define the rules of accessing the Function using APIs.

5. Go to **Discovery and Network > API Rules** and choose **Create**.

6. Fill in the form in the **Create API Rule** view using the following details and choose **Create**.

    - **Name**: for example, `hello-rule`
    - **Service Name**: `hello-world`
    - **Port**: `80`
    - Leave the pre-defined details in the **Gateway** section
    - **Host**: for example, `hello-host`
    - **Access Strategy**: `No Auth`

7. The `hello-rule` APIRule is created. Scroll down to **Virtual Service**, copy the URL under **Hosts**, and paste it in your browser.

    A browser window opens showing the following result:

     **`Hello World from the Kyma Function hello-world running on nodejs20!`**

### Deploy a microservice in Kyma

You already know how to deploy and expose a Function. You can do the same with a container microservice.

To deploy a microservice, you will use the **orders-service** example. For more information about this example, see the [kyma-runtime-extension-samples](https://github.com/SAP-samples/kyma-runtime-extension-samples/tree/main/orders-service) repository.

This tutorial shows how to deploy a microservice using the Docker image.

1. Open Kyma dashboard, go to **Namespaces**, and choose **default**.

2. Go to **Workloads > Deployments** and choose **Create**.

3. Fill in the form in the **Create Deployments** view using the following details and then choose **Create**.

    - **Name**: `orders-deployment`
    - Switch the toggle to enable Istio sidecar proxy injection for the Deployment.
    - **Docker Image**: `ghcr.io/sap-samples/kyma-runtime-extension-samples/orders-service:1.0.0`
    - Optionally, provide the following parameters to save resources:

    | Profile | Value | Profile | Value |
    | :--- | ---: | :--- | ---: |
    | Memory requests | 10Mi | Memory limits   | 32Mi |
    | CPU requests (m) | 16m | CPU limits (m)  | 20m  |

You created the **orders-deployment** Deployment. To confirm that the operation was successful, check if the **Status** field is showing **1/1** Pods running.

### Create a Service

Once you have the Deployment ready, you can create a Kubernetes Service to allow other Kubernetes resources to communicate with your microservice.

1. Go to **Discovery and Network > Services** and choose **Create**.

2. Fill in the form in the **Create Service** view using the following details and then choose **Create**.

    - **Name**: `orders-service`
    - Add the following selector:
        - **app**: `orders-deployment`
    - Add a port using the following parameters:
        - **Name**: `orders-port`
        - **Protocol**: `TCP`
        - **Port**: `80`
        - **Target Port**: `8080` (or other)
        - **Application Protocol**: `http`

You created a new Service, called **orders-service**.

### Expose the microservice

You cannot access and test your new `orders-service` yet from outside of the cluster. To expose the microservice, first, you must create an **APIRule**, just like you did to expose your Function.

1. Go to **Discovery and Network > API Rules** and choose **Create**.

2. Fill in the form in the **Create API Rule** view using the following details and choose **Create**.

    - **Name**: `orders-apirule`
    - **Service Name**: `orders-service`
    - **Port**: `80`
    - Leave the pre-defined details in the **Gateway** section
    - **Host**: `orders-host`
    - **Methods**: choose `GET` and `POST`
    - **Access Strategy**: `No Auth`

3. The `orders-apirule` is created. Wait for the **Status** to be `Ready`.

4. Scroll down to **Virtual Service**, copy the URL under **Hosts**, and paste it in your browser. A browser will open, but the link doesn't work because the underlying Docker image has no homepage. Extend the URL by adding the **/orders** string.

    For example:
    `https://orders-host.c-123456.kyma.ondemand.com/orders`

5. If you extend the URL correctly, you can see your orders: **`[]`**. It is empty, as you have not created orders in this tutorial so far.

### Create sample content for your Service

1. In the terminal call your service using curl. Replace `${APP_URL}` with your `orders-host` URL, for example, `https://orders-host.b1234567.kyma.ondemand.com/`.

    ```bash
    curl -X GET ${APP_URL}/orders -k
    ```

    The result should be still **`[]`**.

2. Place an order using curl replacing the `${APP_URL}` with your `orders-host` URL:

    > If you use Windows, use a Linux-like bash, for example, Git Bash to be able to copy and paste the sample code.

    ```bash
    curl -X POST ${APP_URL}/orders -k \
      -H "Content-Type: application/json" -d \
      '{
          "consignmentCode": "76272727",
          "orderCode": "76272725",
          "consignmentStatus": "PICKUP_COMPLETE"
      }'
    ```

3. Call your `orders-service` in your browser again. The `orders-service` returns the order:

    `[{"orderCode":"76272725","consignmentCode":"76272727","consignmentStatus":"PICKUP_COMPLETE"}]`

    For a complete guide on how to run the `orders-service`, see [the example repository in GitHub](https://github.com/SAP-samples/kyma-runtime-extension-samples/tree/main/orders-service)

Congratulations, you created and exposed your first microservice!
