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
- You have added the Serverless module to your Kyma environment.
- You have completed the [Install the Kubernetes Command Line Tool](https://developers.sap.com/tutorials/cp-kyma-download-cli.html) tutorial.

### Install Kyma CLI

[OPTION BEGIN [macOS]]
1. To install Kyma CLI, run the following command:
```Shell/Bash
brew install kyma-cli
```
2. Check if the installation is successful:
```Shell/Bash
kyma version --client
```
You should see a version number.
[OPTION END]

[OPTION BEGIN [Windows]]
You can install Kyma CLI using chocolatey.

1. To install Kyma CLI, run the following command:
```Shell/Bash
choco install kyma-cli
```
2. Check if the installation is successful:
```Shell/Bash
kyma version --client
```
You should see something like:
`Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"windows/amd64"}`
[OPTION END]

### Create a Function

1. In your terminal, go to your working folder, and run:

    ```
    kyma init function --name hello-function
    ```

    You should see the following message:
    `Project generated in {WORKING_FOLDER_PATH}`

    You have now created 3 files in your working folder:

    - config.yaml
    - handler.js
    - package.json

    You can check the content of the files with your editor (e.g. Visual Studio Code).

1. To apply your Function to your Kyma runtime, run:

    ```bash
    kyma apply function
    ```
   
    You should see the following message: 

    ```bash
    Configuration loaded
    Function - hello-function created
    ```
  
2. To verify the Function deployment, run:

    ```bash
    kubectl get functions hello-function
    ```
   
    You should get the following result:

    ```bash
    NAME             CONFIGURED   BUILT   RUNNING   RUNTIME    VERSION   AGE
    hello-function   True         True    True      nodejs20   1         4m9s
    ```
   
### Expose the Function   

1. Run the following command to get the domain name of your Kyma cluster:

    ```bash
    kubectl get gateway -n kyma-system kyma-gateway \
            -o jsonpath='{.spec.servers[0].hosts[0]}'
    ```


2. In your working folder, create a new YAML file (e.g. `myapirule.yaml`), and add the following API rule definition:

    ```yaml
    apiVersion: gateway.kyma-project.io/v2alpha1
    kind: APIRule
    metadata:
      name: hello-rule
      namespace: default
    spec:
      hosts:
        - hello-host.{CLUSTER_DOMAIN}
      service:
        name: hello-function
        port: 80
      gateway: kyma-system/kyma-gateway
      rules:
        - path: /*
          methods: ["GET", "POST"]
          noAuth: true
    ```

3. Replace `{CLUSTER_DOMAIN}` with your cluster domain obtained in Step 1. For example: `hello-host.123456789.kyma.ondemand.com`.

4. Deploy your API Rule. 
   
    ```bash
    kubectl apply -f "{PATH_TO_YOUR_CONFIG_FILE}"
    ```

    For example:   

    ```bash
    kubectl apply -f "C:\tools\myapirule.yaml"
    ```
   
### Verify the Function exposure

1. Run the following curl command, replacing `{CLUSTER_DOMAIN}` with your domain: 

    ```bash
    curl https://hello-host.$CLUSTER_DOMAIN
    ```

    For example:

    ```bash 
    curl https://hello-host.12345678.kyma.ondemand.com/
    ```

If the deployment was successful, you should see the `Hello Serverless` message.
