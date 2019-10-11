# Multi-Tenant Ingress using Istio

## Contents
- [Infrastructure Setup](#infrastructure-setup)
- [Deploy Addtional Ingress Gateway](#deploy-additional-ingress-gateway)
- [Deploy Test Apps](#deploy-test-apps)
- [Setup and Configure HTTPS](#setup-and-configure-https)

## Infrastructure Setup

First, create the GKE cluster:

```bash
gcloud beta container clusters create [CLUSTER_NAME] \
    --machine-type=n1-standard-2 \
    --cluster-version=latest \
    --enable-stackdriver-kubernetes --enable-ip-alias \
    --scopes cloud-platform
```

Grab the cluster credentials - you'll need them for `kubectl` commands to work:

```bash
gcloud container clusters get-credentials [CLUSTER_NAME]
```

Make yourself a `cluster-admin` so you can install Istio:

```bash
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

Next, grab the latest release of Istio:

```bash
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.2 sh -
cd istio-1.3.2
```

Create the `istio-system` namespace:

```bash
kubectl create namespace istio-system
```

Now use `helm` to install the Istio `CustomResourceDefinition`s:

```bash
helm template install/kubernetes/helm/istio-init \
    --name istio-init \
    --set certmanager.enabled=true \
    --namespace istio-system | kubectl apply -f -
```

Confirm that **23** CRDs we're in fact installed:

```bash
kubectl get crds | grep 'istio.io' | wc -l
```

Now use `helm` to install the Istio control plane components, using the `default` installation profile, and also enabling `certmanager`, `kiali`, and `grafana`.

```bash
helm template install/kubernetes/helm/istio \
    --name istio \
    --namespace istio-system \
    --set certmanager.enabled=true \
    --set certmanager.email=[YOUREMAIL@YOURDOMAIN.COM] \
    --set gateways.istio-ingressgateway.sds.enabled=true \
    --set kiali.enabled=true \
    --set grafana.enabled=true | kubectl apply -f -
```

Finally, turn on Istio's auto-injection for the `default` namespace so that all Pods deployed to `default` get the `istio-proxy` automatically injected.

```bash
kubectl label ns default istio-injection=enabled
```

## Deploy Additional Ingress Gateway

Now that Istio is up and running, use the following steps to run additional Istio `ingressgateway` deployments.

Throughout `example-ig-serviceaccount.yaml`, `example-ig-deployment.yaml`, and `example-ig-service.yaml` there are references to `example-ingressgateway`. The objects in these files can be renamed for additional `ingressgateway` deployments but keep in mind, you will have to update values in multiple places. See below for a semi-exhaustive list of the changes.

First, create the `ServiceAccount`:

```bash
kubectl apply -n istio-system -f ingressgateway/example-ig-serviceaccount.yaml
```

Next, create the `Deployment`:

```bash
kubectl apply -n istio-system -f ingressgateway/example-ig-deployment.yaml
```

Finally, expose the `Deployment` using a `Service` (which also provisions a `LoadBalancer`):

```bash
kubectl apply -n istio-system -f ingressgateway/example-ig-service.yaml
```

### Changes from `istio-ingressgateway`

[`example-ig-serviceaccount.yaml`](/ingressgateway/example-ig-serviceaccount.yaml):
- `metadata.name`
- `metadata.labels`

[`example-ig-deployment.yaml`](/ingressgateway/example-ig-deployment.yaml):
- `metadata.name`
- `metadata.labels`
- `spec.selector.matchLabels`
- `spec.template.metadata.labels`
- `spec.containers[].env[].ISTIO_META_WORKLOAD_NAME`
- `spec.containers[].env[].ISTIO_META_OWNER`
- `spec.containers[].name`
- `spec.containers[].volumeMounts[]`
- `spec.serviceAccountName`
- `spec.volumes[]`

[`example-ig-service.yaml`](/ingressgateway/example-ig-service.yaml):
- `metadata.name`
- `metadata.labels`
- `spec.ports[].http2.nodePort`
- `spec.ports[].https.nodePort`
- `spec.ports[].tcp.nodePort`
- `spec.selector`

### Generating Additonal Ingress Gateways

If you need to run more than one `ingressgateway`, you can copy & update the examples found in [`ingressgateway/`](/ingressgateway) or you can use `helm` to generate an `istio-ingressgateway`. You'll need to generate and update three objects: `ServiceAccount`, `Deployment`, and `Service`.

```bash
for TYPE in serviceaccount deployment service; do
    helm template istio-1.3.2/install/kubernetes/helm/istio \
        --name istio --namespace istio-system \
        --execute charts/gateways/templates/$TYPE.yaml \
        --set gateways.istio-ingressgateway.sds.enabled=true \
        >> my-ingressgateway.yaml
done
```

Next, edit the files as needed, updating values that correspond to the required [changes above](#changes-from-istio-ingressgateway).

## Deploy Test Apps

Create a `Namespace` for each app:

```bash
kubectl create ns hello-v1
kubectl create ns hello-v2
```

Activate `istio-proxy` auto-injection for each new `Namespace`:

```bash
kubectl label ns hello-v1 istio-injection=enabled
kubectl label ns hello-v2 istio-injection=enabled
```

Now, deploy the `helloworld` apps and Istio configuration:

```bash
kubectl apply -f apps/helloworld-deployment.yaml
kubectl apply -f apps/hello-v1-networking.yaml
kubectl apply -f apps/hello-v2-networking.yaml
```

Now `helloworld-v1` is running in the `hello-v1` namespace, and `istio-ingressgateway` is configured to send external traffic to that service using a `Gateway`/`VirtualService` pair. 

Similarly, `helloworld-v2` is running in the `hello-v2` namespace, and `example-ingressgateway` is configured to send external traffic to that service using a `Gateway`/`VirtualService` pair. 

## Setup and Configure HTTPS

**TODO**

References:
- [Kubernetes Ingress with Cert-Manager](https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/)
- [Deploy a Custom Ingress Gateway Using Cert-Manager](https://istio.io/blog/2019/custom-ingress-gateway/)
- [Istio 1.1.7 + Cert-Manager + Let's Encrypt](https://medium.com/@prune998/istio-1-1-7-lets-encrypt-working-9100cea9f503)
