---
parser: v2
auto_validation: true
time: 40
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Deploy a Go PostgreSQL API Endpoint in SAP BTP, Kyma Runtime
<!-- description --> Develop and deploy an PostgreSQL API endpoint written in Go to SAP BTP, Kyma runtime.

## Prerequisites
  - [Docker](https://www.docker.com/)
  - [Go ](https://golang.org/doc/install)
  - [Git](https://git-scm.com/downloads)
  - [kubectl configured to kubeconfig downloaded from SAP BTP, Kyma runtime](cp-kyma-download-cli)
  - [Use and Seed SAP BTP PostgreSQL in SAP BTP, Kyma Runtime](cp-kyma-postgres-seed) tutorial completed

## You will learn
  - How to configure and build a Go Docker image
  - How to deploy the Go Docker image to SAP BTP, Kyma runtime

## Intro
This tutorial expects that the tutorial [Use and Seed SAP BTP PostgreSQL in SAP BTP, Kyma Runtime](cp-kyma-postgres-seed) has been completed. The Go API will connect to the BTP-managed PostgreSQL instance using the Service Binding Secret available in the Kyma cluster.

Deploying the image includes:

- A Kubernetes ConfigMap to store the database host configuration
- A Kubernetes Service to expose the Go application to other Kubernetes resources
- A Kyma APIRule to expose the API to the Internet
- References to the PostgreSQL Service Binding Secret for credentials

---

### Clone the Git repository

1. In your browser, go to [kyma-runtime-samples](https://github.com/SAP-samples/kyma-runtime-samples). This repository contains a collection of Kyma sample applications which will be used during the tutorial.
   
2. Use the **Code** button to choose one of the options to download the code locally, or simply run the following command within your CLI at your desired folder location:

    ```Shell/Bash
    git clone https://github.com/SAP-samples/kyma-runtime-samples
    ```

### Explore the sample

1. Open the `api-postgresql-go` directory in your desired editor.

2. Explore the content of the sample.

    Within the `cmd/api` directory, you can find `main.go`, which is the main entry point of the Go application.

    The `docker` directory contains the Dockerfile used to generate the Docker image. The image is built in two stages to create an image with a small file size.
    
    In the first stage, a Go image is used. It copies the related content of the project into the image and builds the application. The built application is then copied into the Docker `scratch` image and exposed on port 8000. The `scratch` image is an empty image containing no other tools within it, so obtaining a shell/bash session is not possible. If desired, the following lines could be commented out to build an image with more included tools, but this results in a larger image.

    ```Shell/Bash
    FROM scratch
    WORKDIR /app
    COPY --from=builder /app/api-postgres-go /app/
    ```

    The `internal` directory contains the rest of the Go application, broken down into three packages: `api`, `config`, and `db`. You can explore their contents to understand the structure and functionality.

    Within the `k8s` directory you can find the Kubernetes/Kyma resources you will apply to your SAP BTP, Kyma runtime.

    Within the root, you can find `go.mod` and `go.sum` files that are used to manage the dependencies the application uses.

### Build the Docker image

Run the following commands from the `api-postgresql-go` directory in your CLI.

Make sure to replace the value of `<your-docker-id>` with your Docker account ID.

> If you're using any device with a non-x86 processor (e.g. MacBook M1/M2) you need to instruct Docker to use x86 images by setting the **DOCKER_DEFAULT_PLATFORM** environment variable using the command `export DOCKER_DEFAULT_PLATFORM=linux/amd64`. Check [Environment variables](https://docs.docker.com/engine/reference/commandline/cli/#environment-variables) for more information.

1. To build the Docker image, run this command:

    ```Shell/Bash
    docker build -t <your-docker-id>/api-postgresql-go -f docker/Dockerfile .
    ```

2. To push the Docker image to your Docker repository, run this command:

    ```Shell/Bash
    docker push <your-docker-id>/api-postgresql-go
    ```

### Apply resources to SAP BTP, Kyma runtime

You can find the resource definitions in the `k8s` folder. If you performed any changes in the configuration, these files may also need to be updated. The folder contains the following files that are relevant to this tutorial:

- `apirule.yaml`: defines the API endpoint which exposes the application to the Internet. This endpoint does not define any authentication access strategy and should be disabled when not in use. 
- `authorizationpolicy.yaml`: allows internal traffic to the service api-postgresql-go in the `dev` namespace.
- `configmap.yaml`: defines the name of the database.
- `deployment.yaml`: defines the deployment definition for the Go API, as well as a service used for communication. This definition references the PostgreSQL Service Binding Secret (`postgres-binding`) for connection details and `configmap.yaml` for the database name.  

1. Within the `deployment.yaml`, adjust the value of `spec.template.spec.containers.image`, commented with **#change it to your image**, to use your Docker image. Also ensure the Secret name matches your PostgreSQL Service Binding (`postgres-binding`). Apply the ConfigMap and Deployment:

    ```Shell/Bash
    kubectl -n dev apply -f ./k8s/configmap.yaml
    kubectl -n dev apply -f ./k8s/deployment.yaml
    ```

2. Verify the status of the Pod by running:

    ```Shell/Bash
    kubectl -n dev get po
    ```

    The Pod should now be running.

    ```Shell/Bash
    NAME                               READY   STATUS    RESTARTS   AGE
    api-postgresql-go-c694bc847-tkthc   2/2     Running   0          23m
    ```

5. Run the following command to get the domain name of your Kyma cluster:

    ```bash
    kubectl get gateways.networking.istio.io -n kyma-system kyma-gateway \
        -o jsonpath='{.spec.servers[0].hosts[0]}'
    ```

    The result looks like this:

    ```bash
    *.<xyz123>.kyma.ondemand.com
    ```

6. Copy the result without the leading `*.`.

7. In `apirule.yaml`, modify `allowOrigins.regex`, to match your domain, and add the `exact: http://localhost:8080` key-value pair. For example:

    > **Note:** The command output uses `*.` as a glob wildcard prefix, but in the `regex` field you replace it with `.*` — the equivalent regex pattern that matches any character sequence. For example, `*.<xyz123>.kyma.ondemand.com` becomes the regex `.*xyz123.kyma.ondemand.com`.

    ```yaml
      ... 
        allowOrigins:
          - regex: ".*xyz123.kyma.ondemand.com"
          - exact: http://localhost:8080
      ...
    ```

    > `exact: http://localhost:8080` is only required if you want to locally test the frontend of your application as part of the [Deploy the SAPUI5 Frontend in SAP BTP, Kyma Runtime](https://developers.sap.com/tutorials/cp-kyma-frontend-ui5-postgres.html) tutorial.

8. Apply the APIRule:

    ```Shell/Bash
    kubectl -n dev apply -f ./k8s/apirule.yaml
    ```

9. Apply the AuthorizationPolicy:

    ```Shell/Bash
    kubectl -n dev apply -f ./k8s/authorizationpolicy.yaml
    ```


### Open the API endpoint

To access the API, use the APIRule you created in the previous step.

1. Open Kyma dashboard.

2. From the menu, choose **Namespaces**.

3. Choose the `dev` namespace.

4. From the menu, choose **Discovery and Network > API Rules**.

5. Choose the **Host** entry for the **api-postgresql-go** APIRule to open the application in the browser, which will produce a **404** error. Append `/orders` to the end of the URL and refresh the page to successfully access the API. The URL should be similar to:

    `https://api-postgresql-go.<cluster>.kyma.ondemand.com/orders`

    >You can use a tool such as `curl` to test the various HTTP methods of the API.

---