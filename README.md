# IBM Developer Edition chart subscriptions with IBM Multicloud Manager

IBM hosts a number of developer charts on the `ibmcom` repository that can be used by the developer community. This repository contains multiple subscriptions that can be deployed to one or more managed clusters by using a subscription custom resource definition. This repository provides you with a guide for building a foundation for deploying the hosted developer charts by using IBM Multicloud Manager channels and subscriptions.

## Configuring a channel for the `ibmcom` Helm repository
This repository provides you with the YAML content to create and use the following channels and subscriptions:

* Two channels and namespaces:
  1. A `secrets` channel that uses the `vault` namespace. You can use this channel to store and apply secrets and configurations for the different developer edition charts.
  2. An `ibmcom` channel that uses the `ibmcom` namespace. This channel represents the public IBM Helm repository.

* One `developer-editions` namespace:
  1. This namespace is used to hold subscriptions, placement rules, and application resources

* Two placement rules that search for managed clusters:
  1. The `dev-placementrule` rule targets managed clusters with the label `environment: Dev`
  2. The `production-placementrule` rule targets managed clusters with the label `environment: Production`

* One application resource:
  1. This resource associates subscriptions and placement rules by using a labelSelector on `purpose: developer-editions`.

### Commands to set up the channels for use with IBM Developer Edition charts

To set up the channels, run the following commands:
```bash
kubectl apply -f ./channels/1-secret-vault-channel.yaml
kubectl apply -f ./channels/2-ibmcom-helm-channel.yaml
```

### Kubernetes resources created on the hub cluster

Namespaces:
* `vault`
* `developer-editions`
* `ibmcom`

Channels:
* A channel that is called `secrets`.
* A channel that is called `ibmcom`.

Application:
* A `developer-editions` application to group all `developer-editions` subscriptions and placement rules together.

Placement rules:
* `dev-placementrule`
* `production-placementrule`

## Subscribe a managed cluster to an IBM Developer Edition Chart

A subscription monitors the Helm repository for new versions of a chart. If no chart is deployed or if a new version of a deployed chart is found, the chart or the new version of the chart is deployed.

### Example that uses the `mq-adv-server-dev` Helm chart

The `mq-adv-server-dev` chart requires two resources to be deployed. One is the resource that initiates the Helm install, the other is a secret that contains the password to be configured for the IBM MQ Administrators.

### Command to subscribe all managed clusters with the label `environment: Dev`

```bash
kubectl apply -f ./subscriptions/3-mqadvanced-subscription.yaml
```

### Kubernetes resources created on the hub cluster

Deployable:
* An `mq-secret-dev` deployable is created in the `vault/secret` channel. This deployable contains a Kubernetes secret that has the IBM MQ Administrators password set in base64.

Subscriptions:
* An `mq-advanced-server-secret-dev` subscription that delivers the Kubernetes secret that is inside the `mq-secret-dev` deployable to the managed clusters that match the associated placement rule.
* An `mq-advanced-server-dev` subscription that propagates to the managed clusters defined in the placement rule, and then deploys the `mq-adv-server-dev` Helm chart. This subscription also contains Helm `values` overrides for the chart.

### IBM Multicloud Manager subscriptions behind the scenes

* When the subscription is created on the hub cluster with a placement rule, the placement rule is evaluated. All managed clusters that match the placement rule get propagated a copy of the subscription.
* Each propagated subscription on the managed cluster completes a different action
  * The `type: Namespace` subscription subscribes the `valut` namespace on the hub cluster and delivers the Kubernetes secret (IBM MQ Administrator password) by finding the deployable resource in that namespace and extracting the `spec.template`.
  *  The `type: HelmRepo` subscription subscribes to the Helm repository path that was specified in the channel. The subscription identifies the correct Helm release and version in the source repository, and then creates a `HelmRelease` resource on the managed cluster. The `HelmRelease` resource then deploys the Helm release by communicating with Tiller.

## Grant permission for deploying the chart (OpenShift)

Within the `developer-editions` namespace, run the following command on the **managed clusters** to grant the default service account `privileged` permission so that the chart can deploy:
```
oc adm policy add-scc-to-user privileged system:serviceaccount:developer-editions:default
```
