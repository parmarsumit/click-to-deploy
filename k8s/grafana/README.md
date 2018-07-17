# Overview

Grafana is an open-source platform for data visualization and monitoring. A large number of
supported data sources makes it a universal visualization tool for many popular open source
data collection systems - including Prometheus, InfluxDB, Elasticsearch, MySQL or PostgreSQL.

## About Google Click to Deploy

Popular open stacks on Kubernetes packaged by Google.

# Installation

## Quick install with Google Cloud Marketplace

Get up and running with a few clicks! Install this Grafana app to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
[on-screen instructions](https://console.cloud.google.com/launcher/details/google/grafana).

## Command line instructions

### Prerequisites

#### Set up command-line tools

You'll need the following tools in your development environment:
- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [docker](https://docs.docker.com/install/)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

#### Create a Google Kubernetes Engine cluster

Create a new cluster from the command-line.

```shell
export CLUSTER=marketplace-cluster
export ZONE=us-west1-a

gcloud container clusters create "$CLUSTER" --zone "$ZONE"
```

Configure `kubectl` to talk to the new cluster.

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### Clone this repo

Clone this repo and the associated tools repo.

```shell
gcloud source repos clone google-click-to-deploy --project=k8s-marketplace-eap
gcloud source repos clone google-marketplace-k8s-app-tools --project=k8s-marketplace-eap
```

#### Install the Application resource definition

Do a one-time setup for your cluster to understand Application resource via installing
Application's Custom Resource Definition.

<!--
To do that, navigate to `k8s/vendor` subdirectory of the repository and run the following command:
-->

```shell
kubectl apply -f google-marketplace-k8s-app-tools/crd/*
```

The Application resource is defined by the
[Kubernetes SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps)
community. The source code can be found on
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).

### Install the Application

Navigate to the `grafana` directory.

```shell
cd google-click-to-deploy/k8s/grafana
```

#### Configure the app with environment variables

Choose the instance name and namespace for the app.

```shell
export APP_INSTANCE_NAME=grafana-1
export NAMESPACE=default
```

Configure the container images.

```shell
export IMAGE_GRAFANA="gcr.io/k8s-marketplace-eap/google/grafana:latest"
export IMAGE_GRAFANA_INIT="gcr.io/k8s-marketplace-eap/google/grafana/debian9:latest"
```

The images above are referenced by
[tag](https://docs.docker.com/engine/reference/commandline/tag). It is strongly
recommended to pin each image to an immutable
[content digest](https://docs.docker.com/registry/spec/api/#content-digests).
This will ensure that the installed application will always use the same images,
until you are ready to upgrade.

```shell
for i in "IMAGE_GRAFANA" \
         "IMAGE_GRAFANA_INIT"; do
  repo=`echo ${!i} | cut -d: -f1`;
  digest=`docker pull ${!i} | sed -n -e 's/Digest: //p'`;
  export $i="$repo@$digest";
  env | grep $i;
done
```

#### Expand the manifest template

Use `envsubst` to expand the template. It is recommended that you save the
expanded manifest file for future updates to the application.

```shell
awk 'BEGINFILE {print "---"}{print}' manifest/* \
  | envsubst '$APP_INSTANCE_NAME $NAMESPACE $IMAGE_GRAFANA $IMAGE_GRAFANA_INIT $NAMESPACE' \
  > "${APP_INSTANCE_NAME}_manifest.yaml"
```

#### Apply to Kubernetes

Use `kubectl` to apply the manifest to your Kubernetes cluster.

```shell
kubectl apply -f "${APP_INSTANCE_NAME}_manifest.yaml" --namespace "${NAMESPACE}"
```

#### View the app in the Google Cloud Console

Point your browser to:

```shell
echo "https://console.cloud.google.com/kubernetes/application/${ZONE}/${CLUSTER}/${NAMESPACE}/${APP_INSTANCE_NAME}"
```

# Access Grafana UI

Grafana is exposed in a ClusterP-only service `$APP_INSTANCE_NAME-grafana`. To connect to
Grafana UI, you can either expose a public service endpoint or keep it private, but connect
from you local environment with `kubectl port-forward`.

## Expose Grafana service publicly

To expose Grafana with a publicly available IP address, run the following command:

```shell
kubectl patch svc "$APP_INSTANCE_NAME-grafana" \
  --namespace "$NAMESPACE" \
  -p '{"spec": {"type": "LoadBalancer"}}
```

It takes a while until service gets reconfigured to be publicly available. After the process
is finished, obtain the public IP address with:

```shell
SERVICE_IP=$(kubectl get svc $APP_INSTANCE_NAME-grafana \
  --namespace $NAMESPACE \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://${SERVICE_IP}:3000/"
```

## Forward Grafana port in local environment

As an alternative to exposing Grafana publicly, you can use a local port forwarding. Run the
following command in background:

```shell
kubectl port-forward --namespace ${NAMESPACE} ${APP_INSTANCE_NAME}-grafana-0 3000
```

With the port forwarded locally, you can access Grafana UI with `http://localhost:3000/`.

## Login to Grafana

Grafana is configured to require authentication. To check your username and password, run the
following commands:

```shell
GRAFANA_USERNAME="$(kubectl get secret $APP_INSTANCE_NAME-grafana \
                      --namespace $NAMESPACE \
                      --output=jsonpath='{.data.admin-user}' | base64 --decode)"
GRAFANA_PASSWORD="$(kubectl get secret $APP_INSTANCE_NAME-grafana \
                      --namespace $NAMESPACE \
                      --output=jsonpath='{.data.admin-password}' | base64 --decode)"
echo "Grafana credentials:"
echo "- user: ${GRAFANA_USERNAME}"
echo "- pass: ${GRAFANA_PASSWORD}"
```

# Scaling

This installation of Grafana is not intended to be scaled up. Please use it with a single replica
only.

# Backup and restore

## Backup Grafana database

Grafana container stores its stateful data in sqlite database, in a file located at
`/var/lib/grafana/grafana.db`.


To backup the current version of the database file, run the following command:

```shell
kubectl cp $NAMESPACE/$APP_INSTANCE_NAME-grafana-0:var/lib/grafana/grafana.db \
  [YOUR_BACKUP_FILE_PATH]
```

To secure your data, we recommend to upload the backup file to a reliable remote location, like
a Google Cloud Storage bucket.

## Restore the database

To restore the Grafana database from backup we will overwrite the `grafana.db` file with a backup
data and recreate the Grafana server's Pod. Run the following instructions:

```shell
kubectl cp [YOUR_BACKUP_FILE_PATH] \
  $NAMESPACE/$APP_INSTANCE_NAME-grafana-0:var/lib/grafana/grafana.db
kubectl delete pod -n $NAMESPACE $APP_INSTANCE_NAME-grafana-0
```

Now wait a while until the Pod gets recreated and changes its status to `Ready`. At this point,
your backup should be restored. Keep in mind that your username and password are also restored
from backup.