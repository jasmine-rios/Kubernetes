# Chapter 3: Deploying a Kubernetes Cluster

Now that you have succesfully built an application container, the next step is to learn how to transform it into a complete, reliable, scalable distributed system.
To do that, you need a working Kubernetes cluster.
At this point, there are cloud-based kubernetes services in most public clouds that make it easy to create a cluster with a few command-line instruction.
We highly recommend this approach if you are just getting started with Kubernetes.
Even if you are ultimately planning on running Kubernetes. Even if you are ultimately planning on running Kubernetes on bare metal, it's a good way to quickly get started with Kubernetes, learn about Kubernetes itself, and then learn how to install it on physical machines.
Futhermore, managing a Kubernetes cluster is a complicated task in itself, and, for most people, it makes sense to defer this mangement to the cloud--especially when the management service is free in most clouds.

Of course, using a cloud-based solution requires paying for those cloud-based resources as well as having an active network connection to the cloud.
For these reasons, local development can be more attractive, and in that case, the minikube tool provides an easy-to-use way to get a local Kubernetes cluster up and running in a VM on your laptop or desktop.
Though this is a nice option, minikube only creates a single-node cluster, which doesn't quite demonstrate all of the aspects of a complete Kubernetes cluster.
For that reason, we recommend people start with a cloud-based solution, unless it really doesn't work for their situation.
A more recent alternative is to run a Docker-in-Docker cluster, which can spin up a multinode cluster on a single machine.
This project is still in beta, though, so keep in mind that you may encounter unexpected issues.

If you truely insist on starting on bare metal, see the Appendix at the end of this book for instructions for building a cluster from a collection of Raspberry Pi single-board computers.
These instruction use the kubeadm tool and can be adapted to other machines beyond Raspberry Pis.

## Installing Kubernetes on a Public Cloud Provider

This chapter covers installing Kubernetes on the three major cloud providers: the Google Cloud platform, Microsoft Azure, and Amazon Web Services.

If you choose to use a cloud provider to manage Kubernets, you need to install only one of these options; once you have a cluster configured and ready to go, you can skip to "The Kubernetes Client", unless you would prefer to install Kubernetes elsewhere.

### Installing Kubernetes with Google Kubernetes Engine

The Google Cloud Platform (GCP) offers a hosted kubernetes-as-a-service called Google Kubernetes Engine (GKE).
To get started with GKE, you need a Google Cloud Platform account with billing enabled and the gcloud tool installed.

Once you have gcloud installed, set a default zone:

`gcloud config set compute/zone us-west1-a`

Then you can create a cluster:

`gcloud container clusters create kuar-cluster --num-nodes=3`

This will take a few minutes.
When the cluster is ready, you can get credentials for the cluster using:

`gcloud container clusters get-credentials kuar-cluster`

If you run into trouble, you can find the complete instructions for creating a GKE cluster in the Google Cloud Platform documentation.

### Installing Kubernetes with Azure Kubernetes Service

Microsoft Azure offers a hosted Kubernetes-as-a-service as part of the Azure Container Service.
The easist way to get started with Azure Container Service to use the built-in Azure Cloud Shell in the Azure portal.
You can activate the shell by clicking the shell icon in the upper-right toolbar.

The shell has the az tool automatically installed and configured to work with your Azure environment.

Alternatively, you can install the az CLI on your local machine.

When you have the shell up and working, you can run

`az group create --name=kuar --location=westus`

Once the resource group is created, you can get credentials for the cluster with:

`az aks get-credentials --resource-group=kuar --name=kuar-cluster`

If you don't already have the kubectl tool installed, you can install it using:

`az aks install-cli`

You can find complete instructions for installing Kubernetes on Azure in the Azure documentation.

### Installing Kubernetes on Amazon Web Services

Amazon offers a Kubernetes service called Elastic Kubernetes Service (EKS).
The easiest way to create an EKS cluster is via the open source eksctl command-line tool.

Once you have eksctl installed and in your path, you can run the following command to create a cluster:

`eksctl create cluster`

For more details on installation options (such as node size and more), view the help using this command

`eksctl create cluster --help`

The cluster installation includes the right configuration for the kubectl command-line tool.
If you don't already have kubectl installed, follow the instruction in the documentation.

## Installing Kubernetes Locally using MiniKube

If you need a local development experience, or you don't want to pay for cloud resources, you can install a simple single-node cluster using minikube.
Alternatively, if you have already installed Docker Desktop, it comes bundled with a single-machine installation of Kubernetes.

While minikube (or Docker Desktop) is a good simulation of a Kubernetes cluster, it's really intended for local development, learning, and experimentation.
Because it only runs in a VM on a single node, it doesn't provide the reliability of a distrubuted Kubernetes cluster.
In addition, certain features described in this book require integration with a cloud provider.
These feastures are either not available or work in a limited way with minikube.

**NOTE**
You need to have a hypervisor installed on your machine to use minikube.
For Linux and macOS, this is generally VirtualBox.
On Windows, the Hyper-V hypervisor is the default option.
Make sure you install the hypervisor before using minikube.
**EON**

You can find the minikube tool on GitHub.
There are binaries for Linux, macOS, and Windows that you can download.
Once you have the minikube tool installed, you can create a local cluster using
`minikube start`

This will create the local VM, provision Kubernetes, and create a local kubectl configuration that points to the cluster.
As mentioned previously, this cluster only has a single node, so while it is useful, it has some differences with most production deployments of Kubernetes.

When you are done with your cluster, you can stop the VM with:

`minikube stop`

If you want to remove the cluster, you can run:

`minikube delete`

## Running Kubernetes in Docker

A different approach to running a Kubernetes cluster, which has been developed more recently, uses Docker containers to simulate multiple Kubernetees nodes instead of running everything in a virtual machine.
The kind project provides a great experience for launching and managing test clusters in Docker. (kind stands for Kubernetes IN Docker.)
kind is still a work in progress (pre 1.0), but is widely used by those building Kubernetes for fast and easy testing.

Installation instructions for your platform can be found at the kiund site/ Once you get it installed, creating a cluster is as easy as:

```bash
kind create cluster --wait 5m
export KUBECONFIG="$(kind get kubeconfig-path)"
kubectl cluster-info
kind delete cluster
```

## The Kubernetes Client

The offical Kubernetes client is kubectl: a command-line tool for interacting with the Kubernetes API.
kubectl can be used to manage most Kubernetes objects, such as pods, RelicaSets, and Services.
kubectl can be used to explore and verify the overall health of the cluster.

We'll use the kubetcl tool to explore the cluster you just created.

### Checking Cluster Status

The first thing you can do is check the version of the cluster that you are running

`kubectl version`

This will display two different versions: the version of the local kubectl tool, as well as the version of the Kubernetes API server.

**NOTE**

Don't worry if these versions are different.
The Kubernetes tools are backward- and forward-compatibile with different versions of the Kuberntes API as long as you stay within two minor versions for both the tools and the cluster and don't try to use newer features on a older cluster.
Kubernetes follows the semantic versioning specification, where the minor version is the middle number (e.g. the 18 in 1.18.2).
However, you will want to make sure that you are within the supported version skew, which is three versions.
If you are not, you may run into problems

**EON**

Now that we've established that you can communicate with your kubernetes cluster, we'll explore the cluster in more depth.