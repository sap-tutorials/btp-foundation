---
parser: v2
auto_validation: true
time: 30
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Use Gateway API to Expose a Workload in SAP BTP, Kyma Runtime
<!-- description --> Use Gateway API to expose a workload.

## Prerequisites
  - [The Istio module added](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module?locale=en-US&version=Cloud) 
  - [kubectl configured to kubeconfig downloaded from SAP BTP, Kyma runtime](cp-kyma-download-cli)
  - [curl](https://curl.se/)

## You will learn
  - How to install Gateway API CRDs in SAP BTP, Kyma runtime
  - How to create and expose a sample workload using Gateway API

## Intro
Learn how to expose a workload in SAP BTP, Kyma runtime using [Gateway API](https://gateway-api.sigs.k8s.io/). Gateway API allows you to expose workloads using Kubernetes resources like Gateway and HTTPRoute, with Istio integration for managing ingress traffic.

> Exposing an unsecured workload to the outside world is a potential security vulnerability, so be careful. If you want to use this example in a production environment, make sure to secure your workload.

---

### Install Gateway API CustomResourceDefinitions

A Gateway API bundle is a collection of Custom Resource Definitions (CRDs) tied to a specific version of Kubernetes Gateway API. Each release of Gateway API provides two channels, standard and regular, which offer different stability levels. The standard release channel includes all resources that have reached General Availability (GA) or beta status, such as GatewayClass, Gateway, HTTPRoute, and ReferenceGrant. These channels are unrelated to Kyma’s fast and regular channels. The Istio module provided by SAP BTP, Kyma runtime supports the Gateway API CRDs installed from the standard channel.

> If you’ve already installed Gateway API CRDs from the experimental channel, you must delete them before installing Gateway API CRDs from the standard channel.

1. To install Gateway API CustomResourceDefinitions (CRDs) from the standard channel, run the following command:

    ```Shell/Bash
    kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
    { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.1.0" | kubectl apply -f -; }
    ```

### Create a workload

1. Export the name of the namespace in which you want to deploy a sample HTTPBin Service:
   
    ```Shell/Bash
    export NAMESPACE={service-namespace}
    ```

2. Create a namespace with Istio injection enabled and deploy the HTTPBin Service:

    ```Shell/Bash
    kubectl create ns $NAMESPACE
    kubectl label namespace $NAMESPACE istio-injection=enabled --overwrite
    kubectl create -n $NAMESPACE -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
    ```

### Expose the workload

1. Create a Kubernetes Gateway to deploy Istio Ingress Gateway:

    ```Shell/Bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: httpbin-gateway
      namespace: ${NAMESPACE}
    spec:
      gatewayClassName: istio
      listeners:
      - name: http
        hostname: "httpbin.kyma.example.com"
        port: 80
        protocol: HTTP
        allowedRoutes:
          namespaces:
            from: Same
    EOF
    ```
    
    This command deploys the Istio Ingress service in your namespace with the corresponding Kubernetes Service of type LoadBalanced and an assigned external IP address.

2. Create an HTTPRoute to configure access to your workload:

    ```Shell/Bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: httpbin
      namespace: ${NAMESPACE}
    spec:
      parentRefs:
      - name: httpbin-gateway
      hostnames: ["httpbin.kyma.example.com"]
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /headers
        backendRefs:
        - name: httpbin
          namespace: ${NAMESPACE}
          port: 8000
    EOF
    ```

### Access the Workload

To access your exposed workload, follow the steps:

1. Discover Istio Ingress Gateway’s IP and port:

    ```Shell/Bash
    export INGRESS_HOST=$(kubectl get gtw httpbin-gateway -n $NAMESPACE -o jsonpath='{.status.addresses[0].value}')
    export INGRESS_PORT=$(kubectl get gtw httpbin-gateway -n $NAMESPACE -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
    ```

2. Call the service:

    ```Shell/Bash
    curl -s -I -HHost:httpbin.kyma.example.com "http://$INGRESS_HOST:$INGRESS_PORT/headers"
    ```
    If successful, you get the code `200 OK` in response.

    > This task assumes there’s no DNS setup for the httpbin.kyma.example.com host, so the call contains the host header.

---