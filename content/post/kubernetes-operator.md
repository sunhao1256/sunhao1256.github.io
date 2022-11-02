---
title: "Kubernetes Operator"
date: 2022-11-02T11:25:27+08:00
tags: ['kubernetes']
---

# Operator

Operator is equivalent for  Custom Resource definations (crds) with  Custom Resource Controller (crc)

# Custom Resources

> A *resource* is an endpoint in the [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) that stores a collection of [API objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/) of a certain kind; for example, the built-in *pods* resource contains a collection of Pod objects.
>
> A *custom resource* is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. However, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.
>
> Custom resources can appear and disappear in a running cluster through dynamic registration, and cluster admins can update custom resources independently of the cluster itself. Once a custom resource is installed, users can create and access its objects using [kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/), just as they do for built-in resources like *Pods*.

# Custom Controller

>on their own , custom resources let you store and retrieve structured data. When you combine a custom resource with a *custom controller*, custom resources provide a true *declarative API*.
>
>The Kubernetes [declarative API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) enforces a separation of responsibilities. You declare the desired state of your resource. The Kubernetes controller keeps the current state of Kubernetes objects in sync with your declared desired state. This is in contrast to an imperative API, where you *instruct* a server what to do.
>
>You can deploy and update a custom controller on a running cluster, independently of the cluster's lifecycle. Custom controllers can work with any kind of resource, but they are especially effective when combined with custom resources. The [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) combines custom resources and custom controllers. You can use custom controllers to encode domain knowledge for specific applications into an extension of the Kubernetes API.

# User Cases

>CustomResourceDefinitions can be used to implement custom resource types for your Kubernetes cluster. These act like most other Resources in Kubernetes, and may be `kubectl apply`'d, etc.
>
>Some example use cases:
>
>- Provisioning/Management of external datastores/databases (eg. CloudSQL/RDS instances)
>- Higher level abstractions around Kubernetes primitives (eg. a single Resource to define an etcd cluster, backed by a Service and a ReplicationController)

# KubeDB

one of User Cases for Provisioning/Management of external datastores/databases
