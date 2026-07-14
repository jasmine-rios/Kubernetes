# Chapter 15: Service Meshes

Perhaps second to only containers, the term service mesh has become synonymous with cloud native development.
However, just like containers, service mesh is a broad term that encompasses a variety of open source projects as well as commercial products.
Understanding the general role of a service mesh in a cloud native architecture is useful.
This chapter will show you what a service mesh is, how different software projects implement them, and finally (and most importantly) when it makes sense to incorporate a service mesh, versus a less complex architecture, into your application.

**NOTE**
In many abstract cloud native architecture diagrams, it seems that a service mesh is necessary for a cloud native architecture.
This is very much not true.
When considering adopting a service mesh, you have to balance the complexity of adding a new component (generally provided by a third party) to your list of depedencies.
In many cases, it is easier and more reliable to simply depend on the existing Kubernetes resources, if they meet the needs of your application.
**EON**

We have previously discussed other networking primitives in Kubernetes like Services and Ingress.
Given the presence of these networking capabilities in the core of Kubernetes, why is there a need to inject additional capabilities (and complexities) into the networking layer?
Fundamentally it comes down to the needs of the software application that is using these networking primitives.

Networking in the core of Kubernetes is really only aware of the applicaton as a destination.
Both Service and Ingress resources have label selectors that route traffic to a particular set of Pods, but beyond that there is comparatively little in the way of additional capabilities that these resources bring.
As an HTTP load balancer, Ingress goes a little beyond this, but the challenge of defining a common API that fits a wide variety of different existing implementations limits the capabilities in the Ingress API.
How can a truly "cloud native" HTTP-routing API be compatible with load balancers and proxies ranging from bare-metal networking devices through to public cloud APIs that were built without thinking about cloud native development?

In a very real way, the development of service mesh APIs outside the core of Kubernetes is a result of this challenge. 
The Ingress APIs bring HTTP(S) traffic from the outside world into a cloud native application.
Within a cloud native application in Kubernetes, freed the need to be compatible with existing infrastructure, the service mesh APIs provide additional cloud native networking capabilities.
So what are these capabilities?
There are three general capabilities provided by most service mesh implementations:
network encryption and authorization, traffic shaping, and observability.
The following sections look at each of these in turn.

## Encryption and Authentication with Mutal TLS

Encryption of network traffic between Pods is a key component to security in a microservice architecture.
Encryptions provided by Mutal Transport Layer Security, or mTLS, is one of the most popular use cases for a service mesh.
While it is possible for developers to implement this encryption themselves, certificate handling and traffic encryption is complicated and hard to get right.
Leaving the implementation of encryption to individual development teams leads to developers forgetting to add encryption at all, or doing it poorly.
When poorly implemented, encryption can negatively impact both reliability, and, in the worst case, provide no real security.
By contrast, installing a service mesh on your Kubernetes cluster automatically provides encryption to network traffic between every Pod in the cluster.
The service mesh adds a sidecar container to every Pod, which transparently intercepts all network communication.
In addition to securing the communication, mTLS adds identity to the encryption using client certificates so your application securely knows the identity of every network client.

## Traffic Shaping

When you first think about your application design, it is typically a clean diagram with a single box for each microservice or layer in the system (e.g., th)