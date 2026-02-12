---
parser: v2
auto_validation: true
time: 25
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Deploy your CAP Bookstore in SAP BTP, Kyma runtime
<!-- description --> Deploy your CAP Bookstore application in SAP BTP, Kyma runtime. Use PostgreSQL as your database.

## Prerequisites
  - [SAP BTP, Kyma runtime enabled](cp-kyma-getting-started)
  - [Deploy PostgreSQL in SAP BTP, Kyma runtime](cp-kyma-postgres-setup) tutorial completed
  - [kubectl installed](https://kubernetes.io/docs/tasks/tools/)
  - [Docker installed](https://www.docker.com/products/docker-desktop/)

## You will learn
- How to deploy your CAP application to your Kyma environment and use PostgreSQL as your database.

## Intro
In this tutorial you will deploy your CAP Bookstore application to your Kyma environment and use PostgreSQL as your database.

### Configure Helm chart for Kyma

1. Generate the Helm chart structure:

    ```bash
    cds add kyma
    ```

    This will prompt you for your container registry (your Docker Hub username).

This creates:

- **chart/Chart.yaml** – Helm chart metadata
- **chart/values.yaml** – Configuration file
- **containerize.yaml** – Configuration for building and publishing container images

2. Update the `chart/values.yaml` file to add database deployer configuration. Add the following section at the end of the file:

    ```yaml
    postgres_deployer:
      enabled: true
      image:
        repository: cap-kyma-bookstore-postgres-deployer
        tag: latest
      bindings:
        postgres-binding:
          fromSecret: postgres-binding
    ```

3. Update the `srv.bindings` section:

    ```yaml
    srv:
      bindings:
        postgres-binding:
          fromSecret: postgres-binding
      image:
        repository: cap-kyma-bookstore-srv
    ```

4. Add `cap-kyma-bookstore-postgres-deployer` to the `modules` section in the `containerize.yaml` file. Replace `YOUR_DOCKER_HUB_USERNAME` with your actual Docker Hub username, if needed.

    ```yaml
    _schema-version: '1.0'
    repository: 'YOUR_DOCKER_HUB_USERNAME'
    tag: latest
    modules:
      - name: bookshop-srv
        ...
      - name: cap-kyma-bookstore-postgres-deployer
        build-parameters:
          buildpack:
            type: nodejs
            builder: builder-jammy-base
            path: gen/pg
            env:
              BP_NODE_RUN_SCRIPTS: ""
    ```

### Configure database deployer Job template

1. Create directory for custom Helm templates:

    ```bash
    mkdir -p chart/templates
    ```

2. Create `chart/templates/db-deployer-job.yaml`:

    ```yaml
    {{- if .Values.postgres_deployer.enabled }}
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: {{ .Release.Name }}-db-deployer
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: deployer
            image: "{{ .Values.global.image.registry }}/{{ .Values.postgres_deployer.image.repository }}:{{ .Values.postgres_deployer.image.tag | default .Values.global.image.tag }}"
            command: ["/cnb/lifecycle/launcher", "npm", "start"]
            env:
            - name: SERVICE_BINDING_ROOT
              value: /bindings
            {{- range $name, $binding := .Values.postgres_deployer.bindings }}
            volumeMounts:
            - name: {{ $name }}
              mountPath: /bindings/{{ $name }}
              readOnly: true
            {{- end }}
          volumes:
          {{- range $name, $binding := .Values.postgres_deployer.bindings }}
          - name: {{ $name }}
            secret:
              secretName: {{ $binding.fromSecret }}
          {{- end }}
          {{- if .Values.global.imagePullSecret.name }}
          imagePullSecrets:
          - name: {{ .Values.global.imagePullSecret.name }}
          {{- end }}
    {{- end }}
    ```
    This Job template runs once to initialize the database schema and load CSV data.

### Deploy to Kyma

1. Deploy your application to Kyma:

    ```bash
    cds up -2 k8s --namespace cap-bookstore
    ```
    > You might get a warning that your registry server is invalid. If so, simply enter your Docker username again.

2. Create an image pull secret when prompted. 

This command executes the following actions:

- Builds your CAP application (`cds build --production`).
- Generates all necessary Helm templates in `gen/chart/templates/` (including the database deployer Job).
- Creates container images for **srv** and **database deployer** using Cloud Native Buildpacks.
- Pushes images to your container registry (configured in `containerize.yaml`).
- Deploys using Helm to your Kyma cluster.
- Runs the database deployer Job to initialize PostgreSQL with the schema and CSV data.

### Verify the deployment

1. In Kyma dashboard, go to your `cap-bookstore` namespace, and select **API Rules**.
2. Choose `cap-kyma-bookstore-srv` and copy the value of `Hosts` from the **Virtual Service** section.
3. Paste it in your browser and add `odata/v4/catalog/Books` at the end. You should see your books and authors.


**Congratulations!** You have now deployed your CAP application with PostgreSQL in SAP BTP, Kyma runtime.