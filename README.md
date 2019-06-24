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
- Your should have access to at least one Kubernetes cluster.
- You should have an application to deploy.

Optional:

Get yourself a local Kubernetes cluster. Although this is not necessary and you can use public cloud for everything, a local cluster is always 
useful when trying things out because it's faster and costs no money at all. Some options:

- [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)
- [kind](https://github.com/kubernetes-sigs/kind)
- [microk8s](https://microk8s.io/)

You can spin up a `kind` cluster by running `make kind` from the root of this repository.

## Building your application image

To deploy your app to Kubernetes, you will need your app to run inside a container. This means, you will need to create that image somehow. There are solutions to automate this (e.g. https://buildpacks.io/) but you can also handle that yourself.

A very simple example of a Dockerfile for a RubyOnRails application can be found here: [Example RoR Dockerfile](examples/RoR_Dockerfile).

## Automation

Next thing is to automate the creation of your image whenever you push your changes to your master branch. A usual development workflow would look something like this:

```
Commit code to master -> CI runs tests -> CI creates the image -> CI deploys to Kubernetes
```

There are many CI tools out there, with many high quality free and open source options among them. For this guide we are going to use Drone.io.


### TODO

- Describe make targets
- Describe directory structure
- Describe how to consume this repo in a project (submodule or other?)

- Can we use buildpacks to automate the building of images?
