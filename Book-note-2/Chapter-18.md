# Chapter 18: Ingresses

Chapter 17 delved into the purpose and creation of the Service primitive.
Once there's a need to expose the application to external consumers, selecting an appropriate Service type becomes crucial.
The most practical choice often involves creating a Service type `LoadBalancer`.
Such a Service offers load balancing capabilities by assigning an external IP address accessible to consumers outside of the Kubernetes cluster.

However, opting for a `LoadBalancer` Service for each externally reachable application has drawbacks.
In a cloud provider environment, each Service triggers the provisioning of an external load balancer, resulting in increased costs.
Additionally, managing a collection of `LoadBalancer` Service objects can lead to administrative challenges, as a new object must be established for each externally accessible microservice.

To mitigate these issues, the Ingress primitive comes into play, offering a singular, load-balanced entry point to an application stack.
An Ingress possesses the ability to route external HTTP(S) request to one or more Services within the cluster based on an optional, DNS-resolvable host name and URL context path.
This chapter will guide you through the creation and access of an Ingress.

**COVERAGE OF CURRICULUM OBJECTIVES**
This chapter addresses the following curriculum objective:
- Know how to use Ingress controllers and Ingress resources
**EON**

**ACESSING AN INGRESS IN MINIKUBE**
Accessing an Ingress in minikube requires special handling.
Refer to the Kubernetes tutorial "Set up Ingress on Minikube"