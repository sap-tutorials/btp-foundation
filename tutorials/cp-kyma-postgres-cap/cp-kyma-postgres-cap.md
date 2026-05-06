---
parser: v2
auto_validation: true
time: 25
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Deploy a CAP Application with PostgreSQL in SAP BTP, Kyma Runtime
<!-- description --> Deploy PostgreSQL in SAP BTP, Kyma runtime for your CAP Bookstore application.

## Prerequisites
  - [SAP BTP, Kyma runtime enabled](cp-kyma-getting-started)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) installed
  - [Node.js and cds-dk installed](https://cap.cloud.sap/docs/get-started/#node-js-and-cds-dk)
  - [Docker installed](https://www.docker.com/)
  - Sufficient quota to add PostgreSQL entitlements in your environment

## You will learn
- How to add and configure PostgreSQL to your Kyma environment.
- How to deploy a CAP application in your Kyma environment.

## Intro
In this tutorial, you will add PostgreSQL entitlements to your subaccount and deploy a CAP Application in the Kyma environment.

### Add PostgreSQL entitlements

1. From your subaccount overview in the SAP BTP cockpit, go to **Entitlements** and choose **Edit**.

2. Choose **Add Service Plans** and search for **PostgreSQL, Hyperscaler Option**.
   
3. Select **free**, and choose **Add 1 Service Plan**.

    > **NOTE**: The available service plans depend on your account configuration. If the **free** plan is not available, select the plan that matches your subaccount entitlements.

4. Choose **Save**.
 
You are now ready to deploy a PostgreSQL instance in your subaccount.

> For the sake of this tutorial, the chosen entitlement units are sufficient. However, when sizing and configuring your environment, consider the necessary amount of entitlement units (2GB/4GB memory blocks and 5GB storage blocks) to support your architecture. Depending on the hyperscaler, configurations can consume different numbers of entitlement units. See [Sizing](https://help.sap.com/docs/postgresql-on-sap-btp/postgresql-on-sap-btp-hyperscaler-option/sizing?locale=en-US) and [Service Plans and Entitlements](https://help.sap.com/docs/postgresql-on-sap-btp/postgresql-on-sap-btp-hyperscaler-option/service-plans-and-entitlements?locale=en-US). 

### Create a sample project

1. Prepare your sample project:

    ```bash
    cds init bookshop --add sample,nodejs && cd bookshop
    ```

2. Go to `db/schema.cds` and for `descr` change `localized String` to `localized LargeString`.

    ```cds
    entity Books : managed {
      key ID       : Integer;
          author   : Association to Authors @mandatory;
          title    : localized String       @mandatory;
          descr    : localized LargeString;
          genre    : Association to Genres;
          stock    : Integer;
          price    : Price;
          currency : Currency;
    }
    ```

### Configure PostgreSQL support

1. Add PostgreSQL deployment configuration:

    ```bash
    cds add postgres --for production
    ```

    > **TIP**: For more information, see [Add PostgreSQL Deployment Configuration](https://cap.cloud.sap/docs/guides/databases/postgres#add-postgresql-deployment-configuration).

2. In `package.json`, change `[production].model` from `db/hana` to `db/postgres` and add the `auth` section:

    ```json
    "cds": {
      "requires": {
        "db-ext": {
          "[development]": {
            "model": "db/sqlite"
          },
          "[production]": {
            "model": "db/postgres"
          }
        },
        "[production]": {
          "db": "postgres"
        },
        "auth": {
          "[production]": {
            "kind": "dummy"
          }
        }
      }
    }
    ```

    > With the dummy authentication, you pass all the authorization checks. It's intended only for development and should not be used in production environments. For more information, see [Dummy Authentication](https://cap.cloud.sap/docs/node.js/authentication#dummy).

### Configure Helm chart for Kyma deployment

1. Generate the Helm chart structure:

    ```bash
    cds add kyma
    ```

2. When prompted, provide your registry username and cluster domain, for example:
    - registry server: user123
    - cluster domain: abc123.kyma.ondemand.com

    > **TIP**: To get your Kyma cluster domain, run:
    > ```
    > kubectl get gateways.networking.istio.io -n kyma-system kyma-gateway \
    > -o jsonpath='{.spec.servers[0].hosts[0]}'
    > ```

    This creates:

    - **chart/Chart.yaml** – Helm chart metadata
    - **chart/values.yaml** – Configuration file
    - **containerize.yaml** – Configuration for building and publishing container images

3. In your `values.yaml` file, change `postgres-db-deployer` to `postgres-deployer` and change `postgres.servicePlanName` to match the service plan you added in the entitlements step.

    ```yaml
    ...
    postgres-deployer:
      image:
        repository: bookshop-postgres-deployer
      bindings:
        postgres:
          serviceInstanceName: postgres
    ...
    postgres:
      serviceOfferingName: postgresql-db
      servicePlanName: free
    ...
    ```

    > **NOTE**: The `servicePlanName` value must match the service plan available in your account. Replace `free` with the plan you selected in the entitlements step if different.

4. If you want to connect to the PostgreSQL instance during local development, add your static internet IP address to the `allow_access` parameter in `values.yaml`. The Kyma NAT IP Gateway is already added by `cds add kyma`. Append your static IP after a comma:

    ```yaml
    postgres:
      serviceOfferingName: postgresql-db
      servicePlanName: free
      parameters:
        allow_access: "<kyma-nat-ip-gateway>,<your-static-internet-ip-address>"
    ```

    > **TIP**: To find your public IP address, you can run `curl ifconfig.me` in your terminal.

### Deploy to Kyma

1. Create the `cap-bookstore` namespace and enable `istio`:

    ```bash
    kubectl create namespace cap-bookstore
    kubectl label namespaces cap-bookstore istio-injection=enabled
    ```

2. Deploy your application to Kyma:

    ```bash
    cds up -2 k8s --namespace cap-bookstore
    ```
    > You might get a warning that your registry server is invalid. If so, simply enter your Docker username again.

3. Create an image pull secret when prompted. 

    This command executes the following actions:

    - Builds your CAP application (`cds build --production`).
    - Generates all necessary Helm templates in `gen/chart/templates/`.
    - Creates container images for **srv** and **database deployer** using Cloud Native Buildpacks.
    - Pushes images to your container registry (configured in `containerize.yaml`).
    - Deploys using Helm to your Kyma cluster.
    - Runs the database deployer Job to initialize PostgreSQL with the schema and CSV data.

    > **NOTE:** The deployment process might timeout, especially during the first deployment when images are being built and pushed. If this happens, the deployment will continue in the background. You can check the status using:
    > ```
    > kubectl get pods -n cap-bookstore
    > ```
    > Wait until all Pods show `Running` status before proceeding to verification.

### Verify the deployment

1. In Kyma dashboard, go to your `cap-bookstore` namespace, and select **API Rules**.
2. Choose `bookshop-srv` and select the link under `Hosts` from the **Status** section.
3. Add `/odata/v4/catalog/Books` at the end of the link. You should see your books and authors.


**Congratulations!** You have now deployed your CAP application with PostgreSQL in SAP BTP, Kyma runtime.