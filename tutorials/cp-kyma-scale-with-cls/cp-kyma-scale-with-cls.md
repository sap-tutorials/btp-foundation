---
parser: v2
auto_validation: true
time: 30
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform ]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Scale Applications Using SAP BTP Cloud Logging Service Metrics
<!-- description --> Configure KEDA to autoscale a Kubernetes workload using application metrics stored in SAP BTP Cloud Logging Service (OpenSearch) as the scaling signal.

## Prerequisites
  - [Get Started with Kyma](https://developers.sap.com/tutorials/cp-kyma-getting-started.html)
  - The Keda, Telemetry, and SAP BTP Operator modules are enabled in your Kyma cluster. See [Enable and Disable a Kyma Module](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module?locale=en-US).
  - Your subaccount has an entitlement for Cloud Logging Service with the `standard` plan. See [Configure Entitlements and Quotas for Subaccounts](https://help.sap.com/docs/btp/sap-business-technology-platform/configure-entitlements-and-quotas-for-subaccounts).
  - `kubectl` is installed and configured to access your Kyma cluster.

## You will learn
  - How to create a Cloud Logging Service instance and expose its OpenSearch API
  - How to deploy a demo application that exposes a custom Prometheus metric
  - How to configure the Kyma Telemetry module to scrape and forward metrics to CLS
  - How to create a KEDA `ScaledObject` that queries CLS (OpenSearch) to drive autoscaling
  - How to observe your workload scaling in response to metric changes

## Intro

SAP BTP Cloud Logging Service (CLS) is a managed observability backend built on OpenSearch. When your application metrics are already flowing into CLS through the Kyma Telemetry module, you can use those same metrics as autoscaling signals without running a separate metrics store.

KEDA 2.20 ships a native `opensearch` scaler that queries CLS directly using an inline query. This tutorial walks you through the full setup: from a demo app that emits a `queue_depth` metric, through Telemetry scraping configuration, to a `ScaledObject` that scales the workload up or down based on that metric's value in CLS.

---

### Create a CLS Instance and Credentials

Choose the setup that matches your deployment strategy.

[OPTION BEGIN [SAP BTP Cockpit]]

1. In the SAP BTP cockpit, go to **Services** → **Instances and Subscriptions** and choose **Create**.

2. Configure the service instance and choose **Next**:
   - **Service**: Cloud Logging
   - **Plan**: standard
   - **Runtime Environment**: Other
   - **Instance Name**: Enter a name, for example `cloud-logging`

3. In the **Parameters** field, add the following parameters to the default JSON and choose **Create**:

    ```json
    {
        "backend": {
            "api_enabled": true,
            "max_data_nodes": 2
        },
        "ingest_otlp": {
            "enabled": true
        }
    }
    ```

4. Wait until the instance status changes to **Created**.

5. In the instance row, choose the **...** (Actions) menu and select **Create Service Binding**. Enter a name for the binding, for example `cloud-logging-binding`, and choose **Create**.

6. Once the binding is created, choose **View Credentials** from the same **...** menu. Note the following values:

   | Key | Description |
   |---|---|
   | **backend-endpoint** | OpenSearch REST API endpoint |
   | **backend-username** | Username for OpenSearch REST API authentication |
   | **backend-password** | Password for OpenSearch REST API authentication |
   | **ingest-otlp-endpoint** | OTLP ingest endpoint for the Telemetry module |
   | **ingest-otlp-cert** | Client certificate for mTLS |
   | **ingest-otlp-key** | Client key for mTLS |

7. Create a namespace and a Kubernetes Secret with the credentials:

    ```bash
    kubectl create namespace cls
    ```

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloud-logging-binding
      namespace: cls
    type: Opaque
    stringData:
      backend-endpoint: "<BACKEND_ENDPOINT>"
      backend-username: "<BACKEND_USERNAME>"
      backend-password: "<BACKEND_PASSWORD>"
      ingest-otlp-endpoint: "<INGEST_OTLP_ENDPOINT>"
      ingest-otlp-cert: |
        <INGEST_OTLP_CERT>
      ingest-otlp-key: |
        <INGEST_OTLP_KEY>
    EOF
    ```

    Replace each placeholder with the corresponding value from the cockpit. For `ingest-otlp-cert` and `ingest-otlp-key`, paste the full PEM content including the `-----BEGIN ...-----` and `-----END ...-----` lines.

[OPTION END]

[OPTION BEGIN [SAP BTP Operator]]

1. Create a namespace for CLS resources and a service instance for Cloud Logging:

    ```bash
    kubectl create namespace cls
    ```

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: services.cloud.sap.com/v1
    kind: ServiceInstance
    metadata:
      name: cloud-logging
      namespace: cls
    spec:
      serviceOfferingName: cloud-logging
      servicePlanName: standard
      parameters:
        backend:
          api_enabled: true
          max_data_nodes: 2
        ingest_otlp:
          enabled: true
    EOF
    ```

2. Wait until the instance is ready:

    ```bash
    kubectl get serviceinstance cloud-logging -n cls -w
    ```

    You should get a result similar to this example:

    ```
    NAME             OFFERING        PLAN       STATUS    AGE
    cloud-logging    cloud-logging   standard   Created    2m
    ```

3. Create a service binding to generate the credentials Secret:

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: services.cloud.sap.com/v1
    kind: ServiceBinding
    metadata:
      name: cloud-logging-binding
      namespace: cls
    spec:
      serviceInstanceName: cloud-logging
    EOF
    ```

4. Verify that the binding Secret was created and contains the required keys:

    ```bash
    kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data}' | jq 'keys'
    ```

    The Secret must contain **backend-endpoint**, **backend-username**, **backend-password**, **ingest-otlp-endpoint**, **ingest-otlp-cert**, and **ingest-otlp-key**.

[OPTION END]

### Deploy the Demo Application

The demo application exposes a Prometheus-format `queue_depth` gauge metric at the `/metrics` endpoint. KEDA uses this value (as stored in CLS) to determine the desired replica count.

The `QUEUE_DEPTH` value is set by an init container at Pod startup. To change the value, update the env var and restart the Pod.

1. Deploy the demo application:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: keda-cls-demo
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: fake-metrics-nginx-config
      namespace: keda-cls-demo
    data:
      default.conf.template: |
        server {
            listen 8080;
            root /usr/share/nginx/html;

            location /metrics {
                default_type "text/plain; version=0.0.4; charset=utf-8";
                try_files /metrics.txt =404;
            }

            location /health {
                default_type text/plain;
                return 200 "OK";
            }

            location / {
                return 404;
            }
        }
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: fake-metrics
      namespace: keda-cls-demo
      labels:
        app: fake-metrics
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: fake-metrics
      template:
        metadata:
          labels:
            app: fake-metrics
        spec:
          initContainers:
          - name: generate-metrics
            image: busybox:1.36
            command:
            - sh
            - -c
            - printf '# HELP queue_depth The current depth of the queue\n# TYPE queue_depth gauge\nqueue_depth %d\n' "$QUEUE_DEPTH" > /data/metrics.txt
            env:
            - name: QUEUE_DEPTH
              value: "10"
            volumeMounts:
            - name: metrics-data
              mountPath: /data
          containers:
          - name: fake-metrics
            image: nginx:alpine
            ports:
            - containerPort: 8080
            resources:
              requests:
                memory: "64Mi"
                cpu: "100m"
              limits:
                memory: "128Mi"
                cpu: "200m"
            volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/templates
            - name: metrics-data
              mountPath: /usr/share/nginx/html
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 10
            readinessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 5
          volumes:
          - name: nginx-config
            configMap:
              name: fake-metrics-nginx-config
          - name: metrics-data
            emptyDir: {}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: fake-metrics
      namespace: keda-cls-demo
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      selector:
        app: fake-metrics
      ports:
      - port: 8080
        protocol: TCP
        targetPort: 8080
      type: ClusterIP
    EOF
    ```

2. Verify that the Pod is running:

    ```bash
    kubectl get pods -n keda-cls-demo
    ```

    You should get a result similar to this example:

    ```
    NAME                           READY   STATUS    RESTARTS   AGE
    fake-metrics-<hash>            1/1     Running   0          30s
    ```

3. Confirm the `/metrics` endpoint is reachable:

    ```bash
    kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never --quiet \
      -- curl -s http://fake-metrics.keda-cls-demo.svc.cluster.local:8080/metrics
    ```

    You should get a result similar to this example:

    ```
    # HELP queue_depth The current depth of the queue
    # TYPE queue_depth gauge
    queue_depth 10
    ```

### Configure the Telemetry Module to Forward Metrics to CLS

The Kyma Telemetry module scrapes Prometheus metrics from annotated Services and forwards them to your CLS instance. The Prometheus scraping annotations are already included in the demo application manifest.

Create a `MetricPipeline` resource that sends the scraped metrics to CLS using the OTLP credentials from the binding Secret:

```bash
kubectl apply -f - <<EOF
apiVersion: telemetry.kyma-project.io/v1beta1
kind: MetricPipeline
metadata:
  name: cls-metric-pipeline
spec:
  input:
    prometheus:
      enabled: true
      namespaces:
        include:
          - keda-cls-demo
    istio:
      enabled: false
    runtime:
      enabled: false
    otlp:
      enabled: true
  output:
    otlp:
      endpoint:
        valueFrom:
          secretKeyRef:
            name: cloud-logging-binding
            namespace: cls
            key: ingest-otlp-endpoint
      tls:
        cert:
          valueFrom:
            secretKeyRef:
              name: cloud-logging-binding
              namespace: cls
              key: ingest-otlp-cert
        key:
          valueFrom:
            secretKeyRef:
              name: cloud-logging-binding
              namespace: cls
              key: ingest-otlp-key
EOF
```

Verify the pipeline is ready:

```bash
kubectl get metricpipeline cls-metric-pipeline
```

You should get a result similar to this example:

```
NAME                  CONFIGURATION GENERATED   GATEWAY HEALTHY   AGENT HEALTHY   FLOW HEALTHY   AGE
cls-metric-pipeline   True                      True              True            True           2m
```

In the CLS OpenSearch Dashboards, confirm that the `queue_depth` metric is arriving:

1. Get the Dashboards URL:

   [OPTION BEGIN [SAP BTP Cockpit]]

   Find the `dashboards-endpoint` value under **View Credentials** for your CLS binding in the SAP BTP cockpit.

   [OPTION END]

   [OPTION BEGIN [SAP BTP Operator]]

   Run the following command:

   ```bash
   echo "https://$(kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data.dashboards-endpoint}' | base64 -d)"
   ```

   [OPTION END]

2. Open the URL in your browser and log in with the Dashboards credentials from your CLS binding.

3. In the navigation menu, go to **Discover**, select the `metrics-otel-v1-*` index pattern, and filter for documents with `name: queue_depth`. The metric should appear within 1-2 minutes after the MetricPipeline becomes healthy.

### Create a KEDA TriggerAuthentication for CLS

KEDA must authenticate with the CLS OpenSearch REST API to run queries. Store the CLS credentials in a Kubernetes Secret and reference them from a `TriggerAuthentication`.

The credentials are available in the CLS service binding Secret. The following keys are used:

| Key | Description |
|---|---|
| **backend-endpoint** | OpenSearch REST API endpoint |
| **backend-username** | Username for OpenSearch REST API authentication |
| **backend-password** | Password for OpenSearch REST API authentication |

1. Create a Secret with your CLS OpenSearch credentials:

    ```bash
    kubectl create secret generic cls-keda-auth \
      --from-literal=username=$(kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data.backend-username}' | base64 -d) \
      --from-literal=password=$(kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data.backend-password}' | base64 -d) \
      -n keda-cls-demo
    ```

2. Create a `TriggerAuthentication` that references the Secret:

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: keda.sh/v1alpha1
    kind: TriggerAuthentication
    metadata:
      name: cls-trigger-auth
      namespace: keda-cls-demo
    spec:
      secretTargetRef:
        - parameter: username
          name: cls-keda-auth
          key: username
        - parameter: password
          name: cls-keda-auth
          key: password
    EOF
    ```

### Create the ScaledObject

The `ScaledObject` tells KEDA to query CLS for the latest `queue_depth` value and scale the demo application accordingly.

1. Export the OpenSearch endpoint from your CLS service binding Secret:

    ```bash
    export CLS_OPENSEARCH_ENDPOINT=https://$(kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data.backend-endpoint}' | base64 -d)
    export CLS_OPENSEARCH_USERNAME=$(kubectl get secret cloud-logging-binding -n cls -o jsonpath='{.data.backend-username}' | base64 -d)
    ```

2. Create the `ScaledObject`:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: keda.sh/v1alpha1
    kind: ScaledObject
    metadata:
      name: cls-queue-depth-scaler
      namespace: keda-cls-demo
    spec:
      scaleTargetRef:
        name: fake-metrics
      minReplicaCount: 1
      maxReplicaCount: 10
      triggers:
        - type: opensearch
          metadata:
            addresses: "${CLS_OPENSEARCH_ENDPOINT}"
            username: "${CLS_OPENSEARCH_USERNAME}"
            index: "metrics-otel-v1-*"
            query: |
              {
                "size": 0,
                "query": {
                  "bool": {
                    "filter": [
                      { "term": { "name": "queue_depth" } },
                      { "range": { "time": { "gte": "now-1m" } } }
                    ]
                  }
                },
                "aggs": {
                  "latest_value": {
                    "max": { "field": "value" }
                  }
                }
              }
            valueLocation: "aggregations.latest_value.value"
            targetValue: "10"
          authenticationRef:
            name: cls-trigger-auth
    EOF
    ```

    > ### Note
    > The `targetValue` of `10` means KEDA targets one replica per 10 units of `queue_depth`. With a `queue_depth` of 42, KEDA targets 5 replicas (`ceil(42/10)`).

3. Verify that KEDA has picked up the scaler:

    ```bash
    kubectl get scaledobject cls-queue-depth-scaler -n keda-cls-demo
    ```

    You should get a result similar to this example:

    ```
    NAME                     SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   TRIGGERS     READY   ACTIVE
    cls-queue-depth-scaler   apps/v1.Deployment   fake-metrics      1     10    opensearch   True    True
    ```

### Observe Autoscaling in Action

1. Check the KEDA-managed HPA:

    ```bash
    kubectl get hpa -n keda-cls-demo
    ```

    You should get a result similar to this example:

    ```
    NAME                              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    keda-hpa-cls-queue-depth-scaler   Deployment/fake-metrics      10/10     1         10        1          2m
    ```

2. Simulate a metric spike by updating the `QUEUE_DEPTH` environment variable and restarting the Pod:

    ```bash
    kubectl set env deployment/fake-metrics QUEUE_DEPTH=80 -n keda-cls-demo
    kubectl rollout restart deployment/fake-metrics -n keda-cls-demo
    ```

    After the next Telemetry scrape and CLS ingestion cycle (typically within 1-2 minutes), KEDA queries CLS and adjusts the replica count.

3. Watch the Pods scale up:

    ```bash
    kubectl get pods -n keda-cls-demo -w
    ```

4. Set the metric back to a lower value to observe scale-down:

    ```bash
    kubectl set env deployment/fake-metrics QUEUE_DEPTH=5 -n keda-cls-demo
    kubectl rollout restart deployment/fake-metrics -n keda-cls-demo
    ```

    After the cooldown period, the replica count returns to the minimum of 1. This may take up to 5 minutes.

### Clean Up

If you want to remove the resources created during this tutorial, run the following commands:

```bash
kubectl delete namespace keda-cls-demo
kubectl delete metricpipeline cls-metric-pipeline
kubectl delete namespace cls
```

[OPTION BEGIN [SAP BTP Operator]]

Also delete the service binding and instance:

```bash
kubectl delete servicebinding cloud-logging-binding -n cls
kubectl delete serviceinstance cloud-logging -n cls
```

[OPTION END]

[OPTION BEGIN [SAP BTP Cockpit]]

Delete the CLS instance in the cockpit under **Services** → **Instances and Subscriptions**.

[OPTION END]
