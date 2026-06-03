---
parser: v2
auto_validation: true
time: 30
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Fast Prototyping in SAP BTP, Kyma Runtime Using App Push
<!-- description --> Deploy a containerized application to SAP BTP, Kyma runtime in a single CLI command using `kyma app push`, with no Dockerfile or external container registry needed.

For this tutorial, we use a Spring Boot application that exposes a REST API for managing movies, storing data in BTP Object Store.

## Prerequisites
 - [SAP BTP, Kyma runtime enabled](cp-kyma-getting-started)
 - [Kyma CLI](https://help.sap.com/docs/btp/sap-business-technology-platform/kyma-cli?locale=en-US#install-kyma-cli) installed
 - [kubectl configured to kubeconfig downloaded from SAP BTP, Kyma runtime](cp-kyma-download-cli)
 - [Git](https://git-scm.com/downloads) installed
 - [Add the Istio, API Gateway, and SAP BTP Operator Kyma modules](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module?locale=en-US#adding-a-kyma-module), if not added by default
- Add the [Docker Registry community module](https://kyma-project.io/external-content/community-modules/docs/user/README.html#quick-install)

## You will learn
  - How to go from source code to a running, externally accessible application on Kyma runtime in a single command
  - How to iterate quickly on a prototype without writing Kubernetes manifests, Dockerfiles, or configuring a container registry
  - How to evolve a local prototype into an automated GitHub Actions CD pipeline

## Intro
In this tutorial, you will deploy a Spring Boot REST API for managing movies, backed by SAP BTP Object Store. The `kyma app push` command builds the application using Cloud Native Buildpacks (no Dockerfile needed), pushes the image to the in-cluster registry, and creates all required Kubernetes resources in one step.

---

### Clone the Git repository

1. Go to the [kyma-runtime-samples](https://github.com/SAP-samples/kyma-runtime-samples) repository and use the green **Code** button to choose one of the options to download the code locally, or simply run the following command using your CLI at your desired folder location:

    ```Shell/Bash
    git clone https://github.com/SAP-samples/kyma-runtime-samples
    ```

### Explore the sample

1. Open the `movies-rest` directory in your desired editor.

2. Explore the content of the sample.

    The directory contains a Maven-based Spring Boot application with REST endpoints for CRUD operations on movie data, backed by SAP BTP Object Store. A `.env` file provides JVM memory tuning required to fit within the default container memory limit.

    > **NOTE:** Object Store is used here for the sake of simplicity. For applications with structured, relational data, use a proper database such as SAP HANA Cloud or PostgreSQL.

### Add Object Store entitlements

1. From your subaccount overview in the SAP BTP cockpit, go to **Entitlements** and choose **Edit**.

2. Choose **Add Service Plans** and search for **Object Store**.

3. Select the **standard** plan and choose **Add 1 Service Plan**.

4. Choose **Save**.

### Create the Object Store ServiceInstance and ServiceBinding

1. Create the `dev` namespace and enable Istio:

    ```Shell/Bash
    kubectl create namespace dev
    ```

    > **NOTE:** Namespaces separate objects inside a Kubernetes cluster. Choosing a different namespace requires adjustments to the provided samples.

2. Create the Object Store ServiceInstance and ServiceBinding:

    ```Shell/Bash
    kubectl -n dev apply -f - <<EOF
    apiVersion: services.cloud.sap.com/v1
    kind: ServiceInstance
    metadata:
      name: object-store-instance
    spec:
      serviceOfferingName: objectstore
      servicePlanName: standard
    ---
    apiVersion: services.cloud.sap.com/v1
    kind: ServiceBinding
    metadata:
      name: object-store-binding
    spec:
      serviceInstanceName: object-store-instance
    EOF
    ```

3. Wait for the binding to become ready:

    ```Shell/Bash
    kubectl -n dev get servicebinding object-store-binding -w
    ```

    Once the `STATUS` column shows `Ready`, a Kubernetes Secret named `object-store-binding` is created in the namespace with the Object Store credentials.

### Deploy the application

1. From the `movies-rest` directory, run the following command to build, push, and deploy the application:

    ```Shell/Bash
    kyma app push \
      --name movies-rest \
      --namespace dev \
      --code-path . \
      --container-port 8080 \
      --expose \
      --istio-inject=true \
      --mount-service-binding-secret object-store-binding \
      --env-from-file .env
    ```

    What happens under the hood:
    - Source code is built into a container image using [Cloud Native Buildpacks](https://buildpacks.io/) (Paketo). No Dockerfile is required — Buildpacks detect `pom.xml` and automatically build a Java application with the correct JDK.
    - The image is pushed to the in-cluster Docker Registry.
    - A Deployment, Service, and APIRule are created.
    - The Object Store binding secret is mounted at `/bindings/secret-object-store-binding`, and `SERVICE_BINDING_ROOT=/bindings` is set automatically.

    > **NOTE:** The same approach works for any language supported by Cloud Native Buildpacks — Node.js, Go, Python, .NET, and more.

### Verify the deployment

1. Once `kyma app push` completes, it prints the app URL:

    ```Shell/Bash
    The movies-rest app is available under the
    movies-rest.<CLUSTER_DOMAIN>.kyma.ondemand.com'
    ```

    > **TIP:** In quiet mode, the app URL is the only output — useful for capturing it in scripts:
    > ```Shell/Bash
    > APP_URL=$(kyma app push ... --quiet)
    > echo $APP_URL
    > ```

2. Open the interactive Swagger UI in your browser at `https://<APP_URL>/swagger-ui.html` and test the CRUD operations on the movies endpoint.

3. Use the Swagger UI to test the CRUD operations on the movies endpoint.

    > **NOTE:** The OpenAPI specification is also available at `https://movies-rest.<CLUSTER_DOMAIN>/v3/api-docs`.

### (Optional) Automate deployments with GitHub Actions

Once your prototype stabilizes, you can automate deployments on every push to your repository using GitHub Actions.

1. Push your application code to a GitHub repository, for example `https://github.com/<YOUR-ORG>/movies-rest`.

2. Authorize the repository's GitHub Actions workflows to deploy to your Kyma cluster:

    ```Shell/Bash
    kyma alpha authorize repository \
      --client-id my-client-id-for-gh-action \
      --cluster-wide \
      --clusterrole edit \
      --repository <YOUR-ORG>/movies-rest
    ```

    This command configures your Kyma cluster to trust GitHub OIDC tokens issued for the specified repository. The workflow will obtain cluster access using a short-lived GitHub OIDC token — no long-lived credentials are stored. The only values you need to keep as secrets are the API server URL and CA certificate, which are connection details rather than credentials.

    > **NOTE:** The `--clusterrole edit` flag is used here for simplicity. In production, choose the most restrictive ClusterRole that satisfies your workflow's needs. You can also limit authorization to a specific workflow, branch, or environment using `--require-claim`. Run `kyma alpha authorize repository --help` for details.

3. Add the cluster connection details as GitHub Actions secrets. Run the following commands locally to get the values:

    ```Shell/Bash
    kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
    kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
    ```

4. In your GitHub repository, go to **Settings** > **Secrets and variables** > **Actions** > **New repository secret** and add:
    - `SERVER` — the API server URL returned by the first command
    - `CA_CRT` — the base64-encoded CA certificate returned by the second command

5. Create the following GitHub Actions workflow file in your repository at `.github/workflows/deploy.yaml`:

    ```yaml
    name: Deploy

    permissions:
      id-token: write
      contents: read

    on:
      push:
        branches:
          - main

    jobs:
      deploy:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4

          - name: Setup Kyma CLI
            uses: kyma-project/setup-kyma-cli@v1.1.0

          - name: Get kubeconfig
            id: oidc
            uses: kyma-project/setup-kyma-cli/kubeconfig@v1.1.0
            with:
              audience: "my-client-id-for-gh-action"
              api-server-url: "${{ secrets.SERVER }}"
              ca-crt: "${{ secrets.CA_CRT }}"
              id-token-auto-refresh: "true"

          - name: Set short SHA
            run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

          - uses: kyma-project/setup-kyma-cli/app-push@v1.1.0
            with:
              name: movies-rest
              namespace: dev
              code-path: .
              build-tag: "${{ env.SHORT_SHA }}"
              container-port: "8080"
              expose: "true"
              istio-inject: "true"
              mount-service-binding-secret: object-store-binding
              kubeconfig: "${{ steps.oidc.outputs.kubeconfig }}"
              env-from-file: .env
              append-output-path: /swagger-ui.html
    ```

6. Every push to the `main` branch now triggers a fresh build and deploy. No local tooling is required after the initial setup.
