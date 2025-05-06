---
parser: v2
auto_validation: true
time: 30
primary_tag: software-product>sap-btp\, kyma-runtime
tags: [ tutorial>beginner, software-product>sap-business-technology-platform]
author_name: Gaurav Abbi, Oliver Stiefbold, Malgorzata Swieca, Jacek Konopelski
keywords: kyma
---

# Create Kyma Function and a Microservice Using Kyma CLI

<!-- description -->In this tutorial, you will use Kyma CLI to create a Kyma Function and a microservice. 

## You will learn

  - How to install and use Kyma CLI
  - How to create a Kyma Function
  - How to create a Kyma microservice

## Prerequisites

- You have created and set up your "SAP BTP, Kyma Environment" either manually or by Quick Account Setup.
- You have [added the Serverless module to your Kyma environment](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module?locale=en-US).
- You have completed the [Install the Kubernetes Command Line Tool](https://developers.sap.com/tutorials/cp-kyma-download-cli.html) tutorial.

### Install Kyma CLI

[OPTION BEGIN [macOS]]
1. To install Kyma CLI, run the following command:

    ```bash
    brew install kyma-cli
    ```
2. Check if the installation is successful:

    ```bash
    kyma version
    ```
You should see a version number.
[OPTION END]

[OPTION BEGIN [Windows]]

1. To install Kyma CLI, go to the [Kyma CLI GitHub release page](https://github.com/kyma-project/cli/releases/tag/3.0.0) and download the relevant Windows binary files.

[OPTION END]

### Create a Function

1. In your terminal, go to your working folder, and run:

    ```bash
    kyma alpha function init
    ```

    You should see the following message:
    `Functions files of runtime nodejs22 initialized to dir {WORKING_FOLDER_PATH}`

    You have now created 2 files in your working folder:

    - handler.js
    - package.json

    You can check the content of the files with your editor (e.g. Visual Studio Code).

2. To apply your Function to your Kyma runtime, run:

    ```bash
    kyma alpha function create hello-function
    ```
   
    You should see the following message: 

    ```bash
    resource default/hello-function applied
    ```
  
3. To verify the Function deployment, run:

    ```bash
    kyma alpha function get hello-function
    ```
   
    You should get the following result:

    ```bash
    NAME             CONFIGURED   BUILT   RUNNING   RUNTIME    GENERATION
    hello-function   True         True    True      nodejs22   1
    ```
   
### Expose the Function   

1. In your working folder, create a new YAML file named `myapirule.yaml`. You can do this using a text editor or by running `touch myapirule.yaml` in your terminal.
   
2. Open the file in your editor and add the following APIRule definition:

    ```yaml
    apiVersion: gateway.kyma-project.io/v2
    kind: APIRule
    metadata:
      name: hello-rule
      namespace: default
    spec:
      hosts:
        - hello-host
      service:
        name: hello-function
        port: 80
      gateway: kyma-system/kyma-gateway
      rules:
        - path: /*
          methods: ["GET", "POST"]
          noAuth: true
    ```

3. Deploy your API Rule. 
   
    ```bash
    kubectl apply -f "{PATH_TO_YOUR_CONFIG_FILE}"
    ```

    For example:   

    ```bash
    kubectl apply -f "C:\tools\myapirule.yaml"
    ```
   
### Verify the Function exposure

1. Run the following command to get the domain name of your Kyma cluster:

    ```bash
    kubectl get gateway -n kyma-system kyma-gateway \
            -o jsonpath='{.spec.servers[0].hosts[0]}'
    ```

    You should see output similar to this:
    
    ```bash
    *.12345678.kyma.ondemand.com
    ```

2. Export the result without the leading `*.` as an environment variable:

    ```bash
    export CLUSTER_DOMAIN={DOMAIN_NAME}
    ```

    For example:

    ```bash
    export CLUSTER_DOMAIN=12345678.kyma.ondemand.com
    ```


3. Run the following curl command, replacing `{CLUSTER_DOMAIN}` with your domain: 

    ```bash
    curl https://hello-host.$CLUSTER_DOMAIN
    ```

    For example:

    ```bash 
    curl https://hello-host.12345678.kyma.ondemand.com/
    ```

If the deployment was successful, you should see the `Hello World!` message.
