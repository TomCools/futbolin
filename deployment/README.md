# Deployment
This document describes how to deploy Futbolín on a Kubernetes cluster.
The guide assumes the use of [Oracle Cloud](https://www.oracle.com/cloud/), but it should work on other Kubernetes environments as well.

## Prerequisites
1. You have an [Oracle Cloud Infrastructure Container Engine for Kubernetes](https://docs.cloud.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengoverview.htm) ready for use.
Make sure that Tiller is enabled in the cluster (see the [../infra folder](../infra) for details).
1. You have [Helm](https://docs.helm.sh/using_helm/#installing-helm) installed.
1. Make sure to use the right Kubernetes context by running `. ./env.sh`.
1. Verify that Tiller is running with `kubectl get pods --namespace kube-system | grep tiller`.
You should see one pod.
1. Create the required namespaces with `kubectl apply -f namespaces.yml`.

## Deploy cert-manager
Following [this guide](https://medium.com/oracledevs/secure-your-kubernetes-services-using-cert-manager-nginx-ingress-and-lets-encrypt-888c8b996260).

* Add the Helm repository that contains cert-manager with `helm repo add jetstack https://charts.jetstack.io`.
* Add the Helm repository that contains nginx with `helm repo add stable https://kubernetes-charts.storage.googleapis.com/`.
* Update the Helm repositories with `helm repo update`.
* Install [cert-manager](https://cert-manager.readthedocs.io/en/latest/index.html) with
```sh
helm install cert-manager \
  --namespace cert-manager \
  --set ingressShim.defaultIssuerName=letsencrypt-staging \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  --set webhook.enabled=false \
  --version v0.13 \
  jetstack/cert-manager
```
* Install the Custom Resource Definitions for cert-manager with `kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.14.0/deploy/manifests/00-crds.yaml`
* Create the Let's Encrypt _Staging_ issuer with `kubectl apply -f letsencrypt-staging.yml`.
This issuer is used to verify everything works.
If it does, we can create a new issuer for obtaining _real_ certificates with `kubectl apply -f letsencrypt-prod.yml`.

### Updating cert-manager
* Ensure the local Helm chart repository cache is up to date with `helm repo update`
* Upgrade cert-manager with the following command (replace with appropriate version number):
```sh
helm repo update
helm upgrade cert-manager \
  --namespace cert-manager \
  --version v0.14 \
  jetstack/cert-manager
```

## Deploy an Nginx ingress
* Update the Helm repositories with `helm repo update`.
* Install the Nginx _ingress_ with
```sh
helm install nginx \
  --set rbac.create=true \
  --namespace kube-system \
  stable/nginx-ingress
```

## Deploy RabbitMQ
* Update the Helm repositories with `helm repo update`.
* Edit rabbitmq-config.yml and fill appropriate values for `rabbitmqPassword` and `rabbitmqErlangCookie`.
  * `rabbitmqPassword` is the password for the RabbitMQ administrative account. The username is `gues`.
  * `rabbitmqErlangCookie` is an [Erlang cookie](https://www.rabbitmq.com/clustering.html#erlang-cookie), a shared secret for nodes to be able to communicate with each other. 
* Deploy a RabbitMQ cluster using the stable [Helm chart](https://github.com/helm/charts/tree/master/stable/rabbitmq-ha):
```sh
helm install rabbitmq \
  -f rabbitmq-config.yml \
  --namespace rabbitmq \
  stable/rabbitmq-ha
```

To show the status of the deployment, issue `helm status rabbitmq --namespace rabbitmq`.

See [the docs](https://github.com/helm/charts/tree/master/stable/rabbitmq-ha) for additional configuration options.
To show information about the deployed release, issue `helm status rabbitmq`.

To upgrade RabbitMQ, read [the upgrade guide](https://github.com/helm/charts/tree/master/stable/rabbitmq-ha#upgrading-the-chart).

## Configuring secrets
Secrets are configured per application component.
See the documentation in each component for the secrets it expects.