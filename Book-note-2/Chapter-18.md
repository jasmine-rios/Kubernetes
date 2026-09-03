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
**EON**

## Working with Ingresses

The Ingress exposes HTTP (and optionally HTTPS) routes to clients outside of the cluster through an externally-reachable URL.
The routing rules configured with the Ingress determine how the traffic should be routed.
Cloud provider Kubernetes environments will often deploy an external load balancer.
The Ingress recieves a public IP address from the load balancer.
You can configure rules for routing traffic to multiple Services based on specific URL context paths.

The sceanrio depicted instantiates an Ingress as the sole entry point for HTTP(S) calls to the domain name next.example.com.
Based on the provided URL context, the Ingress directs the traffic to either of the fictional Services: one designed for a business application and the other for fetching metrics related to the application.

Specifically the URL context path `/app` is routed to the App Service responsible for managing the business application.
Conversely, sending a request to the URL context `/metrics` results in the call being forwarded to the Metrics Service, which is capable of returning relevant metrics.

### Installing an Ingress Controller

For Ingress to function, an Ingress controller is essential.
This controller assesses the set of rules outlined by an Ingress, dictating the routing of traffic.
The choice of Ingress controller often depends on the specific use cases, requirements, and preference of the Kubernetes cluster administrator.
Noteworthy examples of production-grade Ingress controllers include the F5 NGINX Ingress Controller or the AKS Application Gateway Ingress Controller.
Additional options can be explored in the Kubernetes documentation.

With the NGINX Ingress Controller we're using, you should find at least one Pod that runs the Ingress controller after installing it.
This output redners the Pod created by the NGINX Ingress controller residing in the namespace `ingress-nginx`:
```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-qqhrp        0/1     Completed   0          60s
ingress-nginx-admission-patch-56z26         0/1     Completed   1          60s
ingress-nginx-controller-7c6974c4d8-2gg8c   1/1     Running     0          60s
```

Once the Ingress controller Pod transitions into the `Running` status you can assume that the rules defined by Ingress objects will be evaluated.

### Deploying Multiple Ingress Controllers

Certainly, deploying multiple Ingress controllers within a single cluster is a feasible option, especially if a Cloud provider has preconfigured an Ingress controller in the Kubernetes cluster.
The Ingress API introduces the attribute `spec.ingressClassName` to facilitate the selection of a specific controller implementation by name.
To identify all installed Ingress classes, you can use the following command:
```bash
$ kubectl get ingressclasses
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       14m
```

Kubernetes determines the default Ingress class by scanning for the annotation `ingressclass.kubernetes.io/is-default-class: "true"` within all Ingress class objects.
In sceanrios where Ingress objects do not explicitly specify an Ingress class using the attribute `spec.ingressClassName`, they automatically default to the Ingress class marked as the default through this annotation.
This mechanism provides flexibility in managing Ingress classes and allows for a default behavior when no specific class is specified in individual Ingress objects.

While Kubernetes doesn't prevent this configuration, when multiple Ingress classes has the `ingressclass.kubernetes.io/is-default-class: "true"` annotation asssigned, it creates ambiguous behavior.
Ingress resources without an explicit `ingressClassName` will typically fail to create, with validation errors about multiple defaults being found, as the system cannot deterministically choose which controller should handle them.

### Configuring Ingress Rules

When creating an Ingress, you have the flexibility to define one or multiple rules.
Each rule encompasses the specification of an optional host, a set of URL context paths, and the backend responsible for outing the incoming traffic.
This structure allows for fine-grained control over how external HTTP(S) requests are directed with the Kubernetes cluster, catering to different services based on specified conditions.
Table 18-1 descrives the three rules

Table 18-1. Ingress rules
| Type | Example | Description |
|---|---|---|
| An optional host | `next.example.com` | If provided, the rules apply to that host. If no host is defined, all inbound HTTP(S) traffic is handled (e.g., if made through the IP address of the Ingress). |
| A list of paths | `/app` | Incoming traffic must match the host and path to correctly forward the traffic to a Service |
| The backend | `app-service:8080` | A combination of a Service name and port. The Ingress allows access to intra-cluster Services defined as type `ClusterIP`. |

An Ingress controller can optionally define a default backend that is used as a fallback route should none of the configured Ingress rules match.
You can learn more about it in the documentation of the Ingress primtive.

### Creating Ingresses

You can create an Ingress with the imperative `create ingress` command.
The main command-line option you need to provide is `--rule`, which defines the rules in a comma-seperated fashion.
The notation for each key-value pair is `<host>/<path>=<service>:<port>`.
Let's create an Ingress object with two rules:
```bash
$ kubectl create ingress next-app \
  --rule="next.example.com/app=app-service:8080" \
  --rule="next.example.com/metrics=metrics-service:9090"
ingress.networking.k8s.io/next-app created
```

If you look at the output of the `create ingress --help` command, more fine-grained rules can be specified.

**SUPPORT FOR TLS TERMINATION**
Port 80 for HTTP traffic is implied, as we didn't specify a reference to a TLS Secret object.
If you have specified `tls=mysecret` in the rule defintion, then the port 443 would be listed here as well.
For more information on enabling HTTPS traffic, seee the Kubernetes documentation.
The exam does not cover configuring TLS termination for an Ingress.
**EON**

Using a YAML manifest to define Ingress is often more intuitive and preferred by many.
It provides a clearer and more structured wat to express the desired configuration.
An Ingress defined as a YAML manifest is shown in Example 18-1.

Example 18-1. An Ingress defined by a YAML manifest
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: next-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1   1
spec:
  rules:
  - host: next.example.com                            2
    http:
      paths:
      - backend:
          service:
            name: app-service
            port:
              number: 8080
        path: /app
        pathType: Exact
      - backend:                                      3
          service:
            name: metrics-service
            port:
              number: 9090
        path: /metrics
        pathType: Exact
```

1. Assigns an NGINX Ingress-specific annotation for rewriting the URL

2. Defines the rule that maps the `app-service` backend to the URL next.example.com/app

3. Defines the rule that maps the `metrics-service` backend to the URL next.example.com/metrics

The Ingress YAML manifest contains one major difference from the live object representation created by the imperative command: the assignment of an Ingress controller annotation.
Some Ingress controller implementations provide annotation to customize their behavior.
You can find the full list of annotations that come with the NGINX Ingress controller in the cooresponding documentation.

### Defining Path Types

The previous YAML manifest demonstrates one of the options for specifying a path type via the attribute `spec.rules[].http.paths[].pathType`.
The path type defines how an incoming request is evaluated against the declared path.
Table 18-2 indicates the evaluation for incoming requests and their paths.
See the Kubernetes documnetation for a more comprehensive list.

Table 18-2 Ingress path types
| Path type | Rules | Incoming request |
|---|---|---|
| `Exact` | `/app` | Matches `/app` but does not match `/app/test` or `/app/`
| `Prefix` | `/app` | Matches `/app`, `/app/`, and `/application` but does not match `/admin` |

The key distinction between the `Exact` and `Prefix` path type lies in the treatment of trailing slashes.
The `Prefix` path type focuses solely on the provided prefix of a URL context path, allowing it to accomodate requests with URLs that include a trailing slash.
In contrast, the `Exact` path type is more stringent, requiring an exact match of the specified URL context path without considering a trailing slash.

### Listing Ingresses

Listing Ingresses can be achieved with the `get ingress` command.
You willl see some of information you specified when creating the Ingress (e.g., the hosts):
```bash
$ kubectl get ingress
NAME       CLASS   HOSTS              ADDRESS        PORTS   AGE
next-app   nginx   next.example.com   192.168.66.4   80      5m38s
```

The Ingress automatically selected the default Ingress class `nginx` configured by the Ingress controller.
You can find that information under the `CLASS` column.
The value listed under the `ADDRESS` columns is the IP address provided by the external load balancer.

### Rendering Ingress Details

The `describe ingress` command is a valuable tool for obtaining detailed information about an Ingress resource.
It presents the rules in a clear table format, which aids in understanding the routing configurations.
Additionally, when troublshooting, it's essential to pay attention to the additional messages or events.

In the provided output, it's evident that there might be an issue with the Servies named `app-service` and `metrics.service` that are mapped in the Ingress rules.
This discrepancy between the specified services and their existence can lead to routing errors:

```bash
$ kubectl describe ingress next-app
Name:             next-app
Labels:           <none>
Namespace:        default
Address:          192.168.66.4
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host              Path  Backends
  ----              ----  --------
  next.example.com
                    /app       app-service:8080 (<error: endpoints \
                    "app-service" not found>)
                    /metrics   metrics-service:9090 (<error: endpoints \
                    "metrics-service" not found>)
Annotations:        <none>
Events:
  Type    Reason  Age                   From                      ...
  ----    ------  ----                  ----                      ...
  Normal  Sync    6m45s (x2 over 7m3s)  nginx-ingress-controller  ...
```

Futhermore, observing the event log that shows syncing activity by the Ingress controller is crucial.
Any warnings or errors in this log can provide insights into potential issues during the synchronization process.

To address this problem, ensure that the specified Services in the Ingres rules actually exist and are accessible within the Kubernetes cluster.
Additionally, review the event log for any relevant messages that might indicate the cause of the discrepancy.

Let's resolve the issue of not being able to route to the backends configured in the Ingress object.
The following commands create the Pods and Services:
```bash
$ kubectl run app --image=k8s.gcr.io/echoserver:1.10 --port=8080 \
  -l app=app-service
pod/app created
$ kubectl run metrics --image=k8s.gcr.io/echoserver:1.10 --port=8080 \
  -l app=metrics-service
pod/metrics created
$ kubectl create service clusterip app-service --tcp=8080:8080
service/app-service created
$ kubectl create service clusterip metrics-service --tcp=9090:8080
service/metrics-service created
```

Inspecting the Ingress object doesn't show any errors for the configured rules.
If you're now able to see a list of resolvable backends along with the corresponding Pod virtual IP addresses and ports, the Ingress object is correctly configured, and the backends are recognized and accessible:
```bash
$ kubectl describe ingress next-app
Name:             next-app
Labels:           <none>
Namespace:        default
Address:          192.168.66.4
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host              Path  Backends
  ----              ----  --------
  next.example.com
                    /app       app-service:8080 (10.244.0.6:8080)
                    /metrics   metrics-service:9090 (10.244.0.7:8080)
Annotations:        <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    13m (x2 over 13m)  nginx-ingress-controller  Scheduled for sync
```

It's worth coming back to the Ingress details if you experience any issues with routing traffic through an Ingress endpoint.

### Accessing an Ingress

To enable the routing of the incoming HTTP(S) traffic through the Ingress and subsequently to the configured Service. it's critical to set up a DNS entry mapping to the external address. 
This typically involces configuring either an Address record (A record) or a Canonical Name (CNAME) record.
The ExternalDNS project is a valuable tool that can assist in managing these DNS records automatically.

For local testing on a Kubernetes cluster on your machine, follow these steps:

1. Find the IP address of the load balancer used by the ingress.
2. Add the IP address to hostname mapping to your /etc/hosts file.

By adding the IP address to your local /etc/hosts/ file, you will resolve the domain name locally, allowing you to test the behacior of the Ingress without relying on external DNS records:
```bash
$ kubectl get ingress next-app \
  --output=jsonpath="{.status.loadBalancer.ingress[0]['ip']}"
192.168.66.4
$ sudo vim /etc/hosts
...
192.168.66.4   next.example.com
```

You can now send HTTP requests to the backend.
This call matches the `Exact` path rule and therefore returns a HTTP 200 response code from the application:
```bash
$ wget next.example.com/app --timeout=5 --tries=1
--2021-11-30 19:34:57--  http://next.example.com/app
Resolving next.example.com (next.example.com)... 192.168.66.4
Connecting to next.example.com (next.example.com)|192.168.66.4|:80... \
connected.
HTTP request sent, awaiting response... 200 OK
```

This next call uses a URL with a trailing slash.
The Ingress path rule doesn't support this case, and therefore the call doesn't go through.
You will recieve an HTTP 404 response code.
For the second call to work, you'd have to change the path rule to `Prefix`:
```bash
$ wget next.example.com/app/ --timeout=5 --tries=1
--2021-11-30 15:36:26--  http://next.example.com/app/
Resolving next.example.com (next.example.com)... 192.168.66.4
Connecting to next.example.com (next.example.com)|192.168.66.4|:80... \
connected.
HTTP request sent, awaiting response... 404 Not Found
2021-11-30 15:36:26 ERROR 404: Not Found.
```

You can observe the same behavior for the Metrics Service configured with the URL context path `metrics`.
Feel free to try that out as well.

## Summary

The resource type Ingress defines rules for routing cluster external HTTP(S) traffic to one or many Service.
Each rule defines a URL context path to target a Service.
For an Ingress to work, you first naeed to install an Ingress controller.
An Ingress controller periodically evaluates those rules and ensures that they apply to the cluster.
To expose the Ingress, a cloud provider usually stands up an external load balancer that lends an external IP address to the Ingress.

## Exam Essentials

Know the difference between a Service and an Ingress
    An Ingress is not to be confused with a Service.
    The Ingress is meant for routing cluster-external HTTP(S) traffic to one or many Services based on an optional hostname and mandatory path.
    A Service routes traffic to a set of Pods.

Understand the role of an Ingress controller
    An Ingress controller needs to be installed before an Ingress can function properly.
    Without installing an Ingress controller, Ingress rules will have no effect.
    You can choose from a range of Ingress controller implementations, all documented on the Kubernetes documentation page.
    Assume that an Ingress controller will be preinstalled for you in the exam environment.

Practice the definition of Ingress rules
    You can define one or many rules in an Ingress.
    Every rule consists of an optional host, the URL context path, and the Service DNS name and port.
    Try defining more than a single rule and how to access the endpoint.
    You will not have to understand the process for configuring TLS termination for an Ingress--this aspect is covered by the CKS exam.
    