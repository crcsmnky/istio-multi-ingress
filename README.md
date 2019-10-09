# Multi-Tenant Ingress using Istio

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
    --set kiali.enabled=true \
    --set grafana.enabled=true | kubectl apply -f -
```

Finally, turn on Istio's auto-injection for the `default` namespace so that all Pods deployed to `default` get the `istio-proxy` automatically injected.

```bash
kubectl label ns default istio-injection=enabled
```

## Add Additional Ingress Gateways

Now that Istio is up and running, use the following steps to run additional Istio `ingressgateway` deployments.

Throughout `example-ig-serviceaccount.yaml`, `example-ig-deployment.yaml`, and `example-ig-service.yaml` there are references to `example-ingressgateway`. The objects in these files can be renamed for additional `ingressgateway` deployments but keep in mind, you will have to update names in multiple places. See below for a semi-exhaustive list of the changes.

First, create the `ServiceAccount`:

```bash
kubectl apply -n istio-system -f example-ig-serviceaccount.yaml
```

Next, create the `Deployment`:

```bash
kubectl apply -n istio-system  -f example-ig-deployment.yaml
```

Finally, expose the `Deployment` using a `Service` (which also provisions a `LoadBalancer`):

```bash
kubectl apply -n istio-system  -f example-ig-service.yaml
```

### Changes from `istio-ingressgateway`

[`example-ig-serviceaccount.yaml`](/example-ig-serviceaccount.yaml):
- `metadata.name`
- `metadata.labels`

[`example-ig-deployment.yaml`](/example-ig-deployment.yaml):
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

[`example-ig-service.yaml`](/example-ig-service.yaml):
- `metadata.name`
- `metadata.labels`
- `spec.ports[].http2.nodePort`
- `spec.ports[].https.nodePort`
- `spec.ports[].tcp.nodePort`
- `spec.selector`

## Deploy Test Apps

- TODO

## Configure LetsEncrypt Certificates

- TODO
- [Deploy a Custom Ingress Gateway Using Cert-Manager](https://istio.io/blog/2019/custom-ingress-gateway/)
