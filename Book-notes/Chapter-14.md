# Chapter 14: Role-Based Access Control for Kubernetes

At this point, nearly every Kubernetes cluster you encounter has role-based access control (RBAC) enabled.
So you have likely encountered RBAC before.
Perhaps you initially couldn't access your cluster until you used some magical command to add a RoleBinding to map a user to a role.
Even though you may have had some exposure to RBAC, you may not have had a great deal of experience understanding RBAC in Kubernetes, including what it is for and how to use it.

Role-based access control provides a mechanism for restricting both access to and actions on Kubernetes APIs to ensure that only authorized users have access.
RBAC is a critical component to both harden access to the Kubernetes cluster where ou are deploying your application and (possibly more important) prevent unexpected accidents where one person in the wrong namespace mistakenly takes down production when they think they are destroying their test cluster.

**NOTE**

While RBAC can be quite useful in limiting access to the Kubernetes API, it's important to remember that anyone who can run arbitrary code inside of Kubernetes cluster can effectively obtain root privilages on the entire cluster.
There are approaches that you can take to make such attacks harder and more expensive, and a correct RBAC setup is part of this defense.
But if you are focused on hostile multitenant security, RBAC by itself is sufficent to protect you.
You must isolate the Pods running on your cluster to provide effective multitenant security, RBAC by itself is sufficent to protect you.
You must isplate the Pods running in your cluster to provide effective multitenant security.
Generally this is done using hypervisor isolated containers or a container sandbox.
**EON**

Before we dive into the details of RBAC in Kubernetes, it's valuable to have a high-level understanding of RBAC as a concept, as well as authentication and authorization more generally.

Every request to Kubernetes is first authenticated.
Authentication provides the identity of the caller issuing the request.
It could be as simple as saying that the request is unauthenticated, or it could integrate deeply with a pluggable authentication provider (e.g., Azure Active Directory) to establish an identity within that third-party system.
Interestingly enough, Kubernetes does not have a built-in identity store, focusing instead on integrating other identity sources within itself.

Once users have been authenticated, the authorization phase determines whether they are authorized to perform the request.
Authorization is a combination of the identity of the user, the resouce (effectively the HTTP path), and the verb or action the user is attempting to perform.
If the particular user is authorized to perform that action on that resource, then the request is allowed to proceed.
Otherwise, an HTTP 403 error is returned.
Let's dive into this process.

## Role-Based Access Control

To properly manage access in Kuberntes, it's critical to understand how indentity, roles, and role bindings interat to control who can do what with which resources.
At first, RBAC can seem like a challenge to understand, with a series of interconnected, abstract concepts; but once it's understood, you can be confident in your ability to manage cluster access.

### Idenity in Kubernetes

Every request to Kubernetes is assocaited with some identity.
Even a request with no identity is associated with the `system:unauthenticated` group.
Kubernetes makes a distinction between user idneties and serice account idenities.
Service accounts are created and managed by Kuberenetes itself and are generally associated with components running inside the cluster.
User accounts are all other accounts associated with actual users of the cluser, and often include automation like continuous delivery services that run outside the cluster.

Kubernetes uses a generaic interface for authentication providers.
Each of the providers supplies a username and, optionally, the set of groups to which the user belongs.
Kubernetes supports a number of authentication providers, including:

- HTTP Basic Authentication (largely deprecated)
- x509 client certificates
- Static token files on the host
- Cloud authentication providers, such as Azure Active Directory, and AWS Identity and Access Management (IAM)
- Authentication webhooks

While most managed Kubernetes installations configure authentication for you, if you are deploying your own authentication, you will need to configure flags on the Kubernetes API server appropriately.

You should always use different identities for different applicaitons in your cluster.
For example, you should have one identity for your production frontends, a different identity for the production backends, and all production identities should be distinct from development identities.
You should also have differnt identities for different clusters.
All of these identities should be machine identities that are not shared with users.
You can either use Kubernetes Service Accounts for achieving this, or you can use a Pod identity provider supplied by your identity system; for example, Azure Active Directory supplies an open source identity provider for Pods as do other popular identity providers.

#### Understanding Roles and Role Bindings

Identity is just the beginning of authorization in Kubernetes.
Once Kubernetes knows the identity of the request, it needs to determine if the request is authorized for that user.
To achieve this, it uses roles and role bindings.

A role is a set of abstract capabilities.
For example, the `appdev` role might represent the abillity to create Pods and Services.
A role binding is an assignment of a role to one or more identities.
Thus, binding the `appdev` role to the user identity `alice` indicates that Alice has the ability to create Pods and Services.

### Roles and Role Bindings in Kubernetes

In kubernetes, two pairs of related resources represent roles and role bindings.
One pair is scoped to a namespace (Role and RoleBinding), while the other pair is scoped to the cluster (ClusterRole and ClusterRoleBinding).

Let's examine Role and RoleBinding first.
Role resources are namespaced and represent capabilities within that single namespace.
You cannot use namespaced roles for nonnamespaced resources (e.g., CustomResourceDefinitions) and 