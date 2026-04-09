---
parser: v2
time: 25
auto_validation: true
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Use and Seed SAP BTP PostgreSQL in SAP BTP, Kyma Runtime
<!-- description --> Use a BTP-managed PostgreSQL instance with Kyma workloads and seed it with sample data.

## Prerequisites
 - [kubectl configured to kubeconfig downloaded from SAP BTP, Kyma runtime](cp-kyma-download-cli)
 - [Git](https://git-scm.com/downloads) installed

## You will learn
  - How to use a Kyma Service Binding Secret that points to an SAP BTP PostgreSQL instance
  - How to seed the PostgreSQL database with sample schema and data using a Kubernetes Job

## Intro
In this tutorial, you will provision a managed PostgreSQL instance on SAP BTP, bind it to your Kyma workload namespace, configure network access, and seed the database with a sample schema and data using a Kubernetes Job.

---

### Clone the Git repository

1. Go to the [kyma-runtime-samples](https://github.com/SAP-samples/kyma-runtime-samples) repository. This repository contains a collection of Kyma sample applications which will be used during the tutorial.

2. Use the green **Code** button to choose one of the options to download the code locally, or simply run the following command using your CLI at your desired folder location:

    ```Shell/Bash
    git clone https://github.com/SAP-samples/kyma-runtime-samples
    ```

### Explore the sample

1. Open the `database-postgres` directory in your desired editor.

2. Explore the content of the sample.

    Within the `k8s` folder you can find `seed-job.yaml`, which includes a ConfigMap containing the SQL and a Kubernetes Job that runs the `psql` client (image `postgres:15`). The Job expects the PostgreSQL connection details to come from a Service Binding Secret named `postgres-binding`. Adjust the Secret name and key names to match your binding if they differ.

### Add PostgreSQL entitlements

1. From your subaccount overview in the SAP BTP cockpit, go to **Entitlements** and choose **Edit**.

2. Choose **Add Service Plans** and search for **PostgreSQL, Hyperscaler Option**.
   
3. Select **free**, and click **Add 1 Service Plan**.

    > **NOTE**: The available service plans depend on your account configuration. If the **free** plan is not available, select the plan that matches your subaccount entitlements.

4. Choose **Save**.

### Apply ServiceInstance and ServiceBinding manifests

1. Create the `dev` namespace and enable `Istio`:

    ```Shell/Bash
    kubectl create namespace dev
    kubectl label namespaces dev istio-injection=enabled
    ```

    > Namespaces separate objects inside a Kubernetes cluster. Choosing a different namespace requires adjustments to the provided samples.

    > Adding the `istio-injection=enabled` label to the namespace enables `Istio`. `Istio` is the service mesh implementation used by SAP BTP, Kyma runtime.

2. Use the provided `postgres-instance-binding.yaml` manifest to create the PostgreSQL instance and binding:

    > **NOTE**: The `postgres-instance-binding.yaml` file uses `free` as the `servicePlanName`. If the **free** plan is not available in your subaccount, open the file and change the `servicePlanName` value to match the plan you entitled in the previous step.

    ```Shell/Bash
    kubectl -n dev apply -f ./k8s/postgres-instance-binding.yaml
    ```

3. It takes some time for the instance and binding to get created. To see if they are in the `Created` state, run:

    ```Shell/Bash
    kubectl -n dev get serviceinstance postgres-instance
    kubectl -n dev get servicebinding postgres-binding
    ```

    Once ready, both resources show `Created` in the `STATUS` column:

### Ensure PostgreSQL allows access

1. Patch the `ServiceInstance` you applied above (namespace and name may differ) so it includes both your public IP and the Kyma NAT IP:

    ```Shell/Bash
    MY_IP=$(curl -s https://api.ipify.org)
    KYMA_NAT_IPS=$(kubectl --namespace kyma-system get configmap kyma-info -o json | jq -r '.data["cloud.natGatewayIps"]')
    kubectl -n dev patch serviceinstance postgres-instance --type=merge \
      -p "{\"spec\":{\"parameters\":{\"allow_access\":\"${MY_IP},${KYMA_NAT_IPS}\"}}}"
    ```
2. It takes some time before the changes are applied. To see if the instance is updated, run:

    ```Shell/Bash
    kubectl -n dev get serviceinstance postgres-instance
    ```
    
### Seed the PostgreSQL database

1. Apply the ConfigMap and Job to seed the database. Run the following commands from the `database-postgresql` directory using your CLI:

    ```Shell/Bash
    kubectl -n dev apply -f ./k8s/seed-job.yaml
    kubectl -n dev get jobs seed-postgresql
    ```

2. Wait until the Job shows `1/1` in the `COMPLETIONS` column:

    ```
    NAME               COMPLETIONS   DURATION   AGE
    seed-postgresql    1/1           12s        30s
    ```

### Verify the data from inside the cluster

1. Run a temporary Pod that maps the Service Binding Secret keys to `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, and `PGSSLMODE` and executes a query. Replace the Secret name and keys if your binding differs.

    ```Shell/Bash
    kubectl -n dev apply -f - <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: pg-client
    spec:
      restartPolicy: Never
      containers:
      - name: psql
        image: postgres:15
        env:
        - name: PGHOST
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: hostname
        - name: PGPORT
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: port
        - name: PGDATABASE
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: dbname
        - name: PGUSER
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: username
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: password
        - name: PGSSLMODE
          valueFrom:
            secretKeyRef:
              name: postgres-binding
              key: sslmode
              optional: true
        command: ["psql"]
        args: ["-v", "ON_ERROR_STOP=1", "-c", "SELECT order_id, description, created FROM orders;"]
    EOF
    ```

2. Check the Pod logs.

    ```Shell/Bash
    kubectl -n dev logs pod/pg-client
    ```
    You should see a table with two sample orders:

    ```
     order_id | description  |         created
    ----------+--------------+-------------------------
     10000001 | Sample Order 1 | 2024-01-01 00:00:00+00
     10000002 | Sample Order 2 | 2024-01-01 00:00:00+00
    (2 rows)
    ```

3. If you want to delete the Pod, run:

    ```Shell/Bash
    kubectl -n dev delete pod/pg-client
    ```


### Clean up

If you want to remove the seeding assets, run:

```Shell/Bash
kubectl -n dev delete job seed-postgresql
kubectl -n dev delete configmap postgresql-sample-sql
```

The PostgreSQL instance itself remains running on SAP BTP and can now be consumed by your Kyma workloads.