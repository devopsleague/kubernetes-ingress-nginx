# Installation Guide

## Contents

- [Prerequisite Generic Deployment Command](#prerequisite-generic-deployment-command)
  - [Provider Specific Steps](#provider-specific-steps)
    - [Docker for Mac](#docker-for-mac)
    - [minikube](#minikube)
    - [AWS](#aws)
    - [GCE - GKE](#gce-gke)
    - [Azure](#azure)
    - [Bare-metal](#bare-metal)
  - [Verify installation](#verify-installation)
  - [Detect installed version](#detect-installed-version)
- [Using Helm](#using-helm)

## Prerequisite Generic Deployment Command

!!! attention
    The default configuration watches Ingress object from *all the namespaces*.
    To change this behavior use the flag `--watch-namespace` to limit the scope to a particular namespace.

!!! warning
    If multiple Ingresses define different paths for the same host, the ingress controller will merge the definitions.

!!! attention
    If you're using GKE you need to initialize your user as a cluster-admin with the following command:
    ```console
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole cluster-admin \
      --user $(gcloud config get-value account)
    ```

### Provider Specific Steps

There are cloud provider specific yaml files.

#### Docker for Mac

Kubernetes is available in Docker for Mac (from [version 18.06.0-ce](https://docs.docker.com/docker-for-mac/release-notes/#stable-releases-of-2018))

[enable]: https://docs.docker.com/docker-for-mac/#kubernetes

Create a service

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

#### minikube

For standard usage:

```console
minikube addons enable ingress
```

For development:

1. Disable the ingress addon:

```console
minikube addons disable ingress
```

2. Execute `make dev-env`
3. Confirm the `nginx-ingress-controller` deployment exists:

```console
$ kubectl get pods -n ingress-nginx
NAME                                       READY     STATUS    RESTARTS   AGE
nginx-ingress-controller-fdcdcd6dd-vvpgs   1/1       Running   0          11s
```

#### AWS

In AWS we use a Network load balancer (NLB) to expose the NGINX Ingress controller behind a Service of `Type=LoadBalancer`.

##### Network Load Balancer (NLB)

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/deploy.yaml
```

##### TLS termination in the Load Balancer (ELB)

In some scenarios is not possible to terminate TLS in the ingress controller but in the Load Balancer.
For this purpose we provide a template:

1. Download [deploy-tls-termination.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/204739fb6650c48fd41dc9505f8fd9ef6bc768e1/deploy/static/provider/aws/deploy-tls-termination.yaml)

```console
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/204739fb6650c48fd41dc9505f8fd9ef6bc768e1/deploy/static/provider/aws/deploy-tls-termination.yaml
```

2. Change:

- Set the VPC CIDR: `proxy-real-ip-cidr: XXX.XXX.XXX/XX`
- Change the AWS Certificate Manager (ACM) ID `service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX`

3. Deploy the manifests:

```console
kubectl apply -f deploy-tls-termination.yaml
```

##### NLB Idle Timeouts

In some scenarios users will need to modify the value of the NLB idle timeout. Users need to ensure the idle timeout is less than the [keepalive_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout) that is configured for NGINX.
By default NGINX `keepalive_timeout` is set to `75s`.

The default NLB idle timeout will work for most scenarios, unless the NGINX [keepalive_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout) has been modified, in which case `service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout` will need to be modified to ensure it is less than the `keepalive_timeout` the user has configured.

_Please Note: An idle timeout of `3600s` is recommended when using WebSockets._

More information with regards to idle timeouts for your Load Balancer can be found in the [official AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html#connection-idle-timeout).

#### GCE-GKE

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

**Important Note:** proxy protocol is not supported in GCE/GKE

#### Azure

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

#### Bare-metal

Using [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

!!! tip
    For extended notes regarding deployments on bare-metal, see [Bare-metal considerations](./baremetal.md).

### Verify installation

To check if the ingress controller pods have started, run the following command:

```console
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```

Once the operator pods are running, you can cancel the above command by typing `Ctrl+C`.
Now, you are ready to create your first ingress.

### Detect installed version

To detect which version of the ingress controller is running, exec into the pod and run `nginx-ingress-controller version` command.

```console
POD_NAMESPACE=ingress-nginx
POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it $POD_NAME -n $POD_NAMESPACE -- /nginx-ingress-controller --version
```

## Using Helm

NGINX Ingress controller can be installed via [Helm](https://helm.sh/) using the chart from the project repository.
To install the chart with the release name `ingress-nginx`:

```console
helm repo add k8s-ingress-nginx https://kubernetes.github.io/ingress-nginx/
helm install ingress-nginx k8s-ingress-nginx
```

If you are using [Helm 2](https://v2.helm.sh/) then specify release name using `--name` flag

```console
helm repo add k8s-ingress-nginx https://kubernetes.github.io/ingress-nginx/
helm install k8s-ingress-nginx --name ingress-nginx
```

### Detect installed version:

```console
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- /nginx-ingress-controller --version
```