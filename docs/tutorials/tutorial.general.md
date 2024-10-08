# Tutorial: Basic

> **_NOTE:_** [Application Gateway for Containers](https://aka.ms/agc) has been released, which introduces numerous performance, resilience, and feature changes. Please consider leveraging Application Gateway for Containers for your next deployment.

These tutorials help illustrate the usage of [Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) to expose an example Kubernetes service through the [Azure Application Gateway](https://azure.microsoft.com/en-us/services/application-gateway/) over HTTP or HTTPS.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deploy `guestbook` application](#deploy-guestbook-application)
- [Expose services over HTTP](#expose-services-over-http)
- [Expose services over HTTPS](#expose-services-over-https)
  - [Without specified hostname](#without-specified-hostname)
  - [With specified hostname](#with-specified-hostname)
- [Integrate with other services](#integrate-with-other-services)

## Prerequisites

- Installed `ingress-azure` helm chart.
  - [**Greenfield Deployment**](../setup/install.md): If you are starting from scratch, refer to these installation instructions which outlines steps to deploy an AKS cluster with Application Gateway and install application gateway ingress controller on the AKS cluster.
- If you want to use HTTPS on this application, you will need a x509 certificate and its private key.

## Deploy `guestbook` application

The guestbook application is a canonical Kubernetes application that composes of a Web UI frontend, a backend and a Redis database. By default, `guestbook` exposes its application through a service with name `frontend` on port `80`. Without a Kubernetes Ingress Resource the service is not accessible from outside the AKS cluster. We will use the application and setup Ingress Resources to access the application through HTTP and HTTPS.

Follow the instructions below to deploy the guestbook application.

1. Download `guestbook-all-in-one.yaml` from [here](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/guestbook-all-in-one.yaml)
1. Deploy `guestbook-all-in-one.yaml` into your AKS cluster by running

  ```bash
  kubectl apply -f guestbook-all-in-one.yaml
  ```

Now, the `guestbook` application has been deployed.

## Expose services over HTTP

In order to expose the guestbook application we will using the following ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: frontend
            port:
              number: 80
```

This ingress will expose the `frontend` service of the `guestbook-all-in-one` deployment
as a default backend of the Application Gateway.

Save the above ingress resource as `ing-guestbook.yaml`.

1. Deploy `ing-guestbook.yaml` by running:

    ```bash
    kubectl apply -f ing-guestbook.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application should be available. You can check this by visiting the
public address of the Application Gateway.

## Expose services over HTTPS

### Without specified hostname

Without specifying hostname, the guestbook service will be available on all the host-names pointing to the application gateway.

1. Before deploying ingress, you need to create a kubernetes secret to host the certificate and private key. You can create a kubernetes secret by running

    ```bash
    kubectl create secret tls <guestbook-secret-name> --key <path-to-key> --cert <path-to-cert>
    ```

1. Define the following ingress. In the ingress, specify the name of the secret in the `secretName` section.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: guestbook
      annotations:
        kubernetes.io/ingress.class: azure/application-gateway
    spec:
      tls:
        - secretName: <guestbook-secret-name>
      rules:
      - http:
          paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: frontend
                port:
                  number: 80
    ```

    *NOTE:* Replace `<guestbook-secret-name>` in the above Ingress Resource with the name of your secret. Store the above Ingress Resource in a file name `ing-guestbook-tls.yaml`.

1. Deploy ing-guestbook-tls.yaml by running

    ```bash
    kubectl apply -f ing-guestbook-tls.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application will be available on HTTPS.

In order to make the `guestbook` application available on HTTP, annotate the `Ingress` with

```yaml
  appgw.ingress.kubernetes.io/ssl-redirect: "true"
```

Only in this case a HTTP Listener is created in Azure which redirects the visitor to the HTTPS version.

### With specified hostname

You can also specify the hostname on the ingress in order to multiplex TLS configurations and services.
By specifying hostname, the guestbook service will only be available on the specified host.

1. Define the following ingress.
    In the ingress, specify the name of the secret in the `secretName` section and replace the hostname in the `hosts` section accordingly.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: guestbook
      annotations:
        kubernetes.io/ingress.class: azure/application-gateway
    spec:
      tls:
        - hosts:
          - <guestbook.contoso.com>
          secretName: <guestbook-secret-name>
      rules:
      - host: <guestbook.contoso.com>
        http:
          paths:
            - pathType: Prefix
              path: /
              backend:
                service:
                  name: frontend
                  port:
                    number: 80
    ```

1. Deploy `ing-guestbook-tls-sni.yaml` by running

    ```bash
    kubectl apply -f ing-guestbook-tls-sni.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application will be available on both HTTP and HTTPS only on the specified host (`<guestbook.contoso.com>` in this example).
