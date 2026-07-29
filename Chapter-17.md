# Chapter 17: Extending Kubernetes

From the beginning, it was clear that Kubernetes was going to be more than its core set of APIs; Once an application is orchestrated within the cluster, there are countless other useful tools and utilities that can be represented and deployed as API objects in the Kubernetes cluster.
The challenge was how to embrace this explosion of objects and use cases without having an API that sprawled without bound.

To resolve this tension between extended use cases and API sprawl, significant effort was put into making the Kubernetes API extensible.
This extensibility meant that cluster operators could customize their clusters with the additional components that suited their needs.
This extensibility enables people to augment their clusters themselves, consume community developed cluster add-ons, and even develop extensions that are bundled and sold in an ecosystem of cluster add-ons, and even develop extensions that are bundled and sold in an ecosystem of cluster plug-ins.
Extensibility has also given rise to a whole new patterns of managing systems, such as the operator pattern.

Regardless of whether you are bundling your own extensions or consuming operators from the ecosystem, understanding how the Kubernetes API server is extended and how extensions can be built and delivered is a key component to unblocking the complete power of Kubernetes using these extensibility mechanisms, a working knowledge of how they operate is critical to understanding how to build applications in a modern Kubernetes cluster.

## What it Means to Extend Kubernetes

In general, extensions to the Kubernetes API server either add new functionality to a cluster or limit and tweak the ways that users can interact with their clusters.
There is a rich ecosystem of plug-ins that cluster administrators can use to add services and capabilties to their clusters.
It's worth noting that extending the cluster is a very high-privilege thing to do.
It is not a capability that should be extended to arbitrary users or arbitrary code because cluster administrator privileges are required to extend a cluster.
Even cluster administrators should be careful and use diligence when installing third-party tools.
Some extensions, like admission controllers, can be used to view all objects being created in the cluster, and could easily be used as a vector to steal Secrets or run malicious code.
Additionally, extending a cluster makes it different then stock Kubernetes.
When running on multiple clusters, it is very valuable to build tooling to maintain consistency of experience across the clusters, and this includes the extensions that are installed.

## Points of Extensibility

There are many ways to extend Kubernetes, from CustomResourceDefinitions to Container Network Interface plug-ins.
This chapter is going to focus on extending the API server by adding new resource types or admission controller to API requests.
We will not cover CNI/CSI/CRI (Container Network Interface/Container Storage Interface/Container Runtime Interface) extensions, as they are more commonly used by Kubernetes cluster providers rather than by the Kubernetes end users, for whom this book was written.

In addition to admission controllers and API extensions, there are actually a number of ways to "extend" your cluster without ever modifying the API server at all.
These include DaemonSets that install automatic logging and monitoring, tools that scan your services for cross-site scripting (XSS) vulnerabilities, and more.
Before embarking on extending your cluster yourself, however, it's worth considering the landscape of things that are possible within the confines of the existing Kubernetes APIs.

To understand the role of admission controllers and CustomResourceDefinitions, it helps to review the flow of requests through the Kubernetes API server.

Addmission controllers are cllaed prior to the API object being written into the backing storage.
Admission controllers can reject or modify API requests.
Several admission controllers are built into the Kubernetes API server; for example, the limit range admission controller that sets default limits for Pods without them.
Many other systems can use custom admission controllers to auto-inject sidecar containers into all Pods created on the system to enable "auto-magic" experiences.

The other form of extension, which can also be used in conjuction with admission controllers, is custom resources.
With custom resources, whole new API objects are added to the Kubernetes API surface area.
These new API objects can be added to namespaces, are subject to RBAC, and can be accessed with existing tools like `kubectl` as well as via the Kubernetes API.

The following sections describe these Kubernetes extension points in greater detail and give both use cases and hands-on examples of how to extend your cluster.

The first thing to do to create a custom resource is to create a CustomResourceDefinition.
This object is actually a meta-resource; that is, a resource that is the definition of another resource.

As a concrete example, consider defining a new resource to represent load tests in your cluster.
When a new LoadTest Resource is created, a load test is spun up in your Kubernetes cluster and drives traffic to a service.

The first step in creating this new resource is defining it through a CustomResourceDefinition.
An example definition looks as follows:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: loadtests.beta.kuar.com
spec:
  group: beta.kuar.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: loadtests
    singular: loadtest
    kind: LoadTest
    shortNames:
    - lt
```

You can see that this a Kubernetes object like any other.
It has a `metadata` sub-object, and within that sub-object, the resource is named.
However, in the case of custom resources, the name is special.
It has to be the format `<resource-plural>.<api-group>` to ensure that each resource definition is unique in the cluster, because the name of each CustomResourceDefinition has to match this pattern, and no two objects in the cluster can have the same name.
We are thus guaranteed that no two CustomResourceDefinitions define the same resource.

In addition to metadata, the CustomResourceDefinition has a `spec` sub-object.
This is where the resource itself is defined.
In that `spec` object, there is an `apigroup` field that supplies the API group for the resource.
As mentioned previously, it must match the suffix of the CustomResourceDefinition's name.
Additionally, there is a list of versions for the resourcem which includes the name of the version (e.g., `v1`, `v2`, etc.), as well as fields that indicate if that version is used for storing data in the backing storage for the API server.
The `storage` field must be true for only a single version for the resource.
There is also a `scope` field to indicate whether the resource is namespaced (the default is namespaced), and a `names` field that allows for the definition of the singular, plural, and `kind` values for the resource.
It also allows the definition of convenience "short names" for the resource for use in `kubectl` and elsewhere.

Given this definition, you can create the resource in the Kubernetes API server.
But first, to show the true nature of dynamic resource types, try to list our `loadtests` resource using `kubectl`:

`kubectl get loadtests`

You'll see that there is no such resource currently defined.
Now use loadtest-resource.yaml to create this resource:

`kubectl create -f loadtest-resource.yaml`

Then get the `loadtests` resource again:

`kubectl get loadtests`

This time you'll see that there is a LoadTest resource type defined, though there are still no instances of this resource type.
Let's change that by creating a new LoadTest resource.

As with all built-in Kubernetes API objects, you can use YAML or JSON to define a custom resource (in this case our LoadTest).
See the following definition:

```yaml
apiVersion: beta.kuar.com/v1
kind: LoadTest
metadata:
  name: my-loadtest
spec:
  service: my-service
  scheme: https
  requestsPerSecond: 1000
  paths:
  - /index.html
  - /login.html
  - /shares/my-shares/
```

One thing you'll note is that we never defined the schema for the custom resource in the CustomResourceDefinition.
It actually is possible to provide an OpenAPI specification (known previously as Swagger) for a custom resource, but this complexity is generally not worth it for simple resource types.
If you do want to perform validation, you can register a validating admission controller, as descrived in the following sections.

You can now use this loadtest.yaml file to create a resource just like you would with any built-in type:

`kubectl create -f loadtest.yaml`

Now when you list the `loadtests` resource, you'll see the newly created resource:

`kubectl get loadtests`

This may be exiciting, but it doesn't really do anything yet.
Sure, you can use this simple CRUD (Create/Read/Update/Delete)
API to manipulate the data for LoadTest objects, but no actual load tests are created in response to this new API we defined because there is no controller present in the cluster to react and take action when a LoadTest object is defined.
The LoadTest custom resource is only half of the infrastructure needs to add LoadTests to our cluster.
The other half is a piece of code that will continuously monitor the custom resources and create, modify, or delete LoadTests as necessary to implement the API.

Just like the user of the API, the controller interacts with the API server to list LoadTests and watches for any change that might occur.
This interaction between controller and API server is shown.

The code for such a controller can range from simple to complex.
The simplest controllers run a `for` loop and repeatedly poll for new custom objects, and then take actions to create or delete the resources that implement those custom objects (e.g., the LoadTest worker Pods).

However, this polling-based approach is inefficient:
the period of the polling loop adds unnessary latency, and the overhead of polling loops adds unnecessary latency, and the overhead of polling may add unnessary load on the API server.
A more efficient approach is to use the watch API on the API server, which provides a stream of updates when they occur, eliminating both the latency and overhead of polling.
However, using this API correctly in a bug-free way is complicated.
As a result, if you want to use watches, it is highly recommended that you use a well-supported mechanism such as the `Informer` pattern exposed in the client-go library.

Now that we have created a custom resource and implemented it via a controller, we have the basic functionality of a new resource in our cluster.
However, many parts of what it means to be a well-functioning resource are missing.
The two most importnant are validation and defaulting.
Validation is the process of ensuring that LoadTest objects sent to the API server are well formed and can be used to create load tests, while defaulting makes it easier for people to use our resources by providing automatic, commly used values by default.
We'll now cover adding these capabilties to our custom resource.

As mentioned earlier, one option for adding validation is via an OpenAPI specification for our objects. 
This can be useful for basic validation of the presence of required fields or the absence of unknown fields.
A complete OpenAPI tutorial is beyong the scope of this book, but there are lots of resource online, including the complete Kubernetes API specification.

Generally speaking, an API schema is actually insufficent for validation of API objects.
For example in our ``