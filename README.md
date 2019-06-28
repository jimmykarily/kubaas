# Kubaas
_replace PaaS with Kubernetes_

## What

This repository is a collection of scripts, docs and other tools to help a developer
implement a development workflow that makes use of Kubernetes cluster(s) for the
various needs of the development process.

## Why

Because using a PaaS usually makes deployment as easy as a `cf push` or a `git push heroku master`. With Kubernetes it is still quite more complicated than that. On the other hand, everything we might need to deploy our app is available in Kubernetes so all we need is a set of scripts to simplify the process.

## Getting started

Prerequisites:

- You should have `kubectl` available on the machine you are planning to work on.
- You should have `helm` available on the machine you are planning to work on.
- You should have `tiller` running on your cluster. `helm init --upgrade` should get it for you.
- Your should have access to at least one Kubernetes cluster.
- You should have an application to deploy.

Optional:

Get yourself a local Kubernetes cluster. Although this is not necessary and you can use public cloud for everything, a local cluster is always
useful when trying things out because it's faster and costs no money at all. Some options:

- [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)
- [kind](https://github.com/kubernetes-sigs/kind)
- [microk8s](https://microk8s.io/)

You can spin up a `kind` cluster by running `make kind` from the root of this repository. If you want to mount your application code in the application container later, set the APPLICATION_PATH environment variable before you run `make kind` to point to the absolute path of your code directory on your local filesystem. This will mount your code under `/code` in the kind node container.

Install [jq](https://stedolan.github.io/jq/). You will find it extremly useful when trying to automate things to have jq around. It will let you parse output from Kube and extract the needed values.

## Building your application image

To deploy your app to Kubernetes, you will need your app to run inside a container. This means, you will need to create that image somehow. There are solutions to automate this (e.g. https://buildpacks.io/) but you can also handle that yourself.

A very simple example of a Dockerfile for a RubyOnRails application can be found here: [Example RoR Dockerfile](examples/RoR_Dockerfile).

## Automation - CI

Next thing you want to do is to automate the creation of your image whenever you push your changes to your master branch.
A usual development workflow would look something like this:

```
Commit code to master -> CI runs tests -> CI creates the image -> CI deploys to Kubernetes
```

There are many CI tools out there, with many high quality free and open source options among them. For this guide we are going to use [Concourse CI](https://concourse-ci.org/).

We suggest you start with a local deployment of Concourse (on a local k8s cluster maybe?) until you get your pipelines right. Then you can worry about deploying Concourse somewhere permanently. Assuming you run a `kind` cluster and  `kubectl` talks to that cluster, first find your node's IP:

```
nodeip=$(docker inspect $(docker ps | grep kindest/node | awk '{print $1}') | jq -r .[0].NetworkSettings.Networks.bridge.IPAddress)
```

then deploy concourse with:

```
helm install stable/concourse --name concourse --namespace concourse  \
--set web.service.type="NodePort" \
--set web.service.atcNodePort=32000 \
--set concourse.web.externalUrl=http://${nodeip}:32000 \
--set concourse.worker.baggageclaim.driver=overlay
```

TODO: use nginx ingress to route to concourse so we can host multiple apps on our local cluster

Access Concourse at `http://${nodeip}:32000` (default login `test:test`). This guide will try to explain the pipeline definitions used but you might want to have Concourse docs handy as well: https://concourse-ci.org/docs.html.

## Automation - Building the app image

We will now implement the pipeline that builds the application image whenever we commit to the master branch of the application source repository (git). We will assume there is a `Dockerfile` at the root of your application repository that builds the application image.
If that's not the case, tweak the pipeline accordingly.

1. Create a new ssh key to be used by the pipeline to talk to your git repository

   You can follow this guide from GitHub on how to do this:
   https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key

2. Make sure you can pull the repository using this new key.
   For GitHub follow this guide: https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys

3. Copy the sample values.yaml from the examples directory:

```
cp examples/concourse_pipeline/values.yaml.sample examples/concourse_pipeline/values.yaml
```

4. Edit `examples/concourse_pipeline/values.yaml` and add your newly generated (private) key.
   **Make sure you never commit this file on source control!** (it's already in `.gitignore` in this repo).

5. Edit the rest of the values in the `values.yaml` to match yours.

6. Deploy the pipeline:

```
fly -t kind set-pipeline -l examples/concourse_pipeline/values.yaml -p deploy  -c examples/concourse_pipeline/pipeline.yaml
```

The pipeline specifies 2 resources, the application code and the docker image. The task uses the application code to build the image and puses that to the registry.

## Deploying the application

For this guide we will assume our application is a RubyOnRails application although the steps to deploy any application should be almost similar. Our application will need to connect to a database and we are going to deploy that on Kubernetes as well. Deploying a database on Kubernetes means we will have to think about replication, backups and other stuff but let's first focus on getting everything running.
We will create a [Helm](https://helm.sh/) chart for our application and all needed components to simplify the deployment and rollbacks.

_This step has been heavily "inspired" by this page: https://docs.bitnami.com/kubernetes/how-to/deploy-rails-application-kubernetes-helm/_

1. Create the application namespace

We will now create the kubernetes namespace where we will deploy our application.

```
kubectl create namespace app
```

1. Create the registry credentials secret
   Your app image should be in a credentials protected (aka private) registry otherwise everyone would be able to pull your application image with your application code in it. For your cluster to be able to pull that private image, you will need to provide credentials in the form of a secret living in Kubernetes. Follow the guide here for more details: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/ . If you have the `config.json` at hand, create the secret:

```
kubectl create secret --namespace app generic my-registry-credentials --from-file=.dockerconfigjson=path_to_your_config.json --type=kubernetes.io/dockerconfigjson
```

1. Install helm

```
helm init --upgrade
```

1. Install nginx-ingress:

The ingress controller will allow us to deploy multiple applications on the same cluster without us needing to remember any ports or IPs (in local clusters or public cloud). Use this command on your local cluster:

```
helm install stable/nginx-ingress --name ingress-nginx --namespace app --set controller.service.type=NodePort
```

A service should be created for the ingress controller and given a NodePort. You can access all ingresses through that port using the local cluster's IP.

For a public cloud you might want to deploy with something like:

```
helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true --set controller.publishService.enabled=true
```

([Read more for gke](https://cloud.google.com/community/tutorials/nginx-ingress-gke))

1. Create a helm chart

If you were starting from scratch, you would create a helm chart with this command:

```
helm create app-helm
```

For this example we will use the already modified chart in `examples/app-helm`. This chart will create a deployment for the application and a stateful set for PostgreSQL (using the `stable/postgresql` chart). Some additional needed Kubernetes objects will also be created.

1. Deploy the app

```
helm install examples/app-helm/ --name app --namespace app
```

You should be able to access your application on `http://172.17.0.2.nip.io:32080/`
(change the IP address to the one of your `kind` container)

### TODO

- Describe make targets
- Describe directory structure
- Describe how to consume this repo in a project (submodule or other?)

- Can we use buildpacks to automate the building of images?
- Backup database
- Rollbacks (with database)
- How do we run database migrations?
- SSL for the app?
- SSL for concourse?
- Change default password for Concourse? Is it safe to have it publicly accessible?
- Use non-root user for postgresql on production
- [DONE] Use a hostPath volume to mount the code for development, check that we get "live" updates.
- Monitor this issue for gce ingress and cert-manager: https://github.com/jetstack/cert-manager/issues/1666
