# Chapter 7: Service Discovery

Kubernetes is a very dynamic system.
The system is involved in placing pods on nodes, making sure they are up and running, and rescheduling them as needed.
There are ways to automatically change the number of Pods based on load (such as Horizontal Pod autoscaling [see "Autoscaling a ReplicaSet"]).
The API-driven nature of the system encourages others to create higher and higher levels of automation.

While the dynamic nature of Kubernetes makes it easy to run a lot of things, it create problems when it comes to finding those things.
Most of the traditional network infrastructure wasn't build for the level of dynamism that Kubernetes presents.

## What is Service Discovery

The general name for this class of problems and solutions is service discovery.
Servide-discovery tools help solve the problem of finding what processes are listening at which addresses for which services.
A good service-discovery system will enable users to resolve this information quickly and reliably.
A good system is also low-latency; clients are updated soon after the information associated with a service changes.
Finally, a good service-discovery system can store a richer definition of what that service is.
For example, perhaps there are multiple ports associated with the service.

The Domain Name System (DNS) is the traditional system of service discovery on the internet.
DNS is designed for relatively stable name resolution with wide and efficent caching.
It is a great system for the internet but falls short in the dynamic world of Kubernetes.

Unfortunately, many systems (for example, Java, by default) look up a name in DNS directly and never re-resolve it.
This can lead to clients caching stale mapping and talking to the wrong IP.
Even with a short TTL (time-to-live) and a well-behaved client, there is a natural delay between when a name resolution changes and when the client notices.
There are natural limits to the amount and type of information that can be returned in a typical DNS query too.
Things start to break past 20 to 30 address (A) records for a single name.
Service (SRV) record solve some problems, but are often very hard to use.
Finally, the way that clients handle multiple IPs in a DNS record is usuallly to take the first IP address and rely on the DNS server to randomize or round-robin the order of records.
There is no substitute for more purpose-built load balancing.

## The Service Object

Real service discovery in Kubernetes starts with a Service object.
A service object is a way to create a named label selector.
As we will see, the Service object does some other nice things for us too.

Just as the `kubectl run` command is an easy way to create a Kuberenets deployment, we can use `kubectl expose` to create a service.
We'll talk about Deployments in detial in Chapter 10, but for now you can think of a Deployment as an instance of a microservice.
Let's create some deployments and services so we can see how they work:

```bash
$ kubectl create deployment alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:blue \
  --port=8080
$ kubectl scale deployment alpaca-prod --replicas 3
$ kubectl expose deployment alpaca-prod
$ kubectl create deployment bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --port=8080
$ kubectl scale deployment bandicoot-prod --replicas 2
  kubectl expose deployment bandicoot-prod
$ kubectl get services -o wide

NAME             CLUSTER-IP    ... PORT(S)  ... SELECTOR
alpaca-prod      10.115.245.13 ... 8080/TCP ... app=alpaca
bandicoot-prod   10.115.242.3  ... 8080/TCP ... app=bandicoot
kubernetes       10.115.240.1  ... 443/TCP  ... <none>
```

After running these commands, we have three services.
The ones we just created are `alpace-prod` and `bandicoot-prod`.
The `kubernetes` service is automatically created for you so you can find and talk to the Kuberntes API from within the app.

IF we look at the `SELECTOR` column, we see that the `alpace-prod` service simply gives a name to a selector and specifies which ports to talk to for that service.
The `kubectl expose` command will conveniently pull both the label selector and relevant ports (8080, in this case) from the deployment definition.

Furthermore, that service is assigned a new type of virtual IP called a cluster IP. 
This is a special IP address the system will load balance across all of the pods that are identified by the selector.

To interact with services, we are going to port-forward to one of the `alpaca` pods.
Start this command and leave it running in a terminal window.
You can see the port-forward working by acessing the `alpaca` Pod at http://localhost:48858:

```bash
$ ALPACA_POD=$(kubectl get pods -l app=alpaca \
    -o jsonpath='{.items[0].metadata.name}')
$ kubectl port-forward $ALPACA_POD 48858:8080
```

### Service DNS

Because the cluster IP is virtual, it is stable, and it is appropriate to give it a DNS address.
All of the issues around clients caching DNS results no longer apply.
Within a namespace, it is as easy as just using the service name to connect to one of the Pods identified by a service.

Kubernetes provides a DNS service exposed to Pods running in the cluster.
This Kubernetes DNS service was installed as a system component when the cluster was first created.
The DNS service is, itself, managed by Kubernetes and is a great example of Kubernetes building on Kubernetes.
The Kubernetes DNS service provides DNS names for cluster IPs.

You can try this out by expanding the "DNS Query" section on the `kuard` server status pag.
Query the A record for `alpaca-prod`.
The output should look something like this:

```bash
;; opcode: QUERY, status: NOERROR, id: 12071
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;alpaca-prod.default.svc.cluster.local.	IN	 A

;; ANSWER SECTION:
alpaca-prod.default.svc.cluster.local.	30	IN	A	10.115.245.13
```

The full DNS name here is `alpaca-prod.default.svc.cluster.local`.
Let's break this down:

    alpaca-prod
        The name of the service in question
    
    default
        The namespace that this service is in.

    svc
        Reconginizing that this is a service.
        This allows Kubernetes to expose other types of things as DNS in the future.
    
    cluster.local.
        The base domain name for the cluster.
        This is the defauly and what you will see for most clusters.
        Administrators may change this to allow unique DNS names across multiple clusters.

When refering to a service in your own namespace, you can just use the service name (`alpace-prod`).
You can also refer to a service in another namespace with `alpaca-prod.default`.
And, of course, yu can use the fully qualified service name (`alpaca-prod.default.svc.cluster.local`).
Try each of these out in the "DNS Query" section of `kuard`.

### Readiness Checks

Often, when an application first starts up, it isn't ready to handle requests.
There is usually some amount of initalization that can take anyewhere from under a second to several minutes.
One nice thing the Service object does is track which of your pods are ready via a readiness check.
Let's modify our deployment to add a readiness check taht is attached to a Pod, as we discussed in chapter 5:

`kubectl edit deployment/alpaca-prod`

This command will fetch the current version of the `alpaca-prod` deployment and bring it up in an editor.
After you save and quit your editor, it'll write the object back to Kubernetes.
This is a quick way to edit an object without saving it to a YAML file.

Add the following section:

```yaml
spec:
  ...
  template:
    ...
    spec:
      containers:
        ...
        name: alpaca-prod
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2
          initialDelaySeconds: 0
          failureThreshold: 3
          successThreshold: 1
```

This sets up the pods this deployment will create so taht they will get checked for readiness via an HTTP `GET` to `/ready` on port 8080.
This check is done every two seconds starting as soon as the Pod comes up.
If three successive checks file, then the Pod will be considered not ready/
However, if only one check succeeds, the Pod will again be considered ready.

Only ready Pods are sent traffic.

Updating the deployment definition like this will delete and re-create the `alpaca` Pods.
As such, we need to restart our `port-forward` command from earlier:

```bash
$ ALPACA_POD=$(kubectl get pods -l app=alpaca-prod \
    -o jsonpath='{.items[0].metadata.name}')
$ kubectl port-forward $ALPACA_POD 48858:8080
```

Point your browser to http://localhost:48858, and you should see the debug page for that instance of `kuard`.
Expand the "Readiness Probe" section.
You should see this page update every time there is a new rediness check from the system, which should happen every two seconds.

In another terminal windows, start a `watch` commmand to the endpoints for the `alpaca-prod` service.
Endpoints are a lower-level way of finding what a service is sending traffic to and are covered later in this chapter.
The `--watch` option here casues the `kubectl` command to hang around and output any updates.
This is an easy way to see how a Kubernetes object changes over time:

`kubectl get endpoints alpaca-prod --watch`

Now return to your browser and hit the "Fail" link for the rediness check.
You should see that the server is now returning errors with codes in the 500s.
After three of these, this server is removed from the list of endpoints for the service.
Hit the "Succeed" link and notice that after a single readines check, the endpoint is added back.

This readiness check is a way for an overloaded or sick server to signal to the system that it doesn't want to recieve traffic anymore.
This is a great way to implement graceful shutdown.
The server can signal that it no longer wants traffic, wait until existing connections are closed, and then cleanly exit.

Press Ctrl-C to exit out of both the `port-forward` and `watch` commands in your terminals.

## Looking Beyond the Cluster

So far, everything we've covered in this chapter has been about exposing services inside of a cluster.
Oftentimes, the IPs for pods are only reachable from within the cluster.
At some point, we have to allow new traffic in!

The most portable way to do this is to use a feature called NodePorts, which enhance a service even further.
In addition to a cluster IP, the system picks a port (or the user can specify one), and every node in the cluster then forwards traffic to that port to the service.

With this feature, if you can reach any node in the cluster, you can contact a service.
You can use the NodePort without knowing where any of the Pods for that service are running.
This can be integrated with hardware or software load balancers to expose the service further.

Try this out by modifying the `alpaca-prod` service:

`kubectl edit service alpaca-prod`

Change the `spec.type` field to `NodePort`.
You can also do this when creating the service via `kubectl expose` by specifying `--type=NodePort`.
The system will assign a new NodePort.

```bash
$ kubectl describe service alpaca-prod

Name:                   alpaca-prod
Namespace:              default
Labels:                 app=alpaca
Annotations:            <none>
Selector:               app=alpaca
Type:                   NodePort
IP:                     10.115.245.13
Port:                   <unset> 8080/TCP
NodePort:               <unset> 32711/TCP
Endpoints:              10.112.1.66:8080,10.112.2.104:8080,10.112.2.105:8080
Session Affinity:       None
No events.
```

Here we see that the system assigned port 32711 to this service.
Now we can hit any of our cluster nodes on that port to access the service.
If you are sitting on the same network, you can access it directly.
If your cluster is in the cloud someplace, you can use SSH tunneling with something like this:

`ssh <node> -L 8080:localhost:32711`

Now if you point your browser to http://localhost:8080, you will be connected to that service.
Each request that you send to the service will be randomly directed to one of the Pods that implements the service.
Reload the page a few times, and you will see that you are randomly assigned to different Pods.

When you are done, exit the SSH session.

## Load Balancer Integration

If you have a cluster that is configured to integrate with external load balancers, you can use the `LoadBalancer` type.
This builds on the `NodePort` type by additionally configuring the cloud to create a new load balancer and direct it at nodes in your cluster.
Most cloud-based Kubernetes clusters offer load balancer integration, and there are a number of project that implement load balancer integration for common physical laod balancers as well, although these may require more manual integration with your cluster.

Edit the `alpaca-prod` service again (`kubectl edit service alpaca-prod`) and change `spec.type` to `LoadBalancer`.

**NOTE**
Creating a service of type `LoadBalancer` exposes that service to the public internet.
Before you do this, you should make certain that it is something that is secure to be exposed to everyone in the world. 
We will discuss security risks further in this section.
Additionally, Chapters 9 and 20 provide guidance on how to secure your application.
**EON**

If you do a `kubectl get services` right away, you'll see that the `EXTERNAL-IP` column for `alpaca-prod` now says `<pending>`.
Wait a bit and you should see a public address assigned by your cloud.
You can look in the console for your cloud accoutn and see the configuration work that Kubernetes did for you:

```bash
$ kubectl describe service alpaca-prod

Name:                   alpaca-prod
Namespace:              default
Labels:                 app=alpaca
Selector:               app=alpaca
Type:                   LoadBalancer
IP:                     10.115.245.13
LoadBalancer Ingress:   104.196.248.204
Port:                   <unset>	8080/TCP
NodePort:               <unset>	32711/TCP
Endpoints:              10.112.1.66:8080,10.112.2.104:8080,10.112.2.105:8080
Session Affinity:       None
Events:
  FirstSeen ... Reason                Message
  --------- ... ------                -------
  3m        ... Type                  NodePort -> LoadBalancer
  3m        ... CreatingLoadBalancer  Creating load balancer
  2m        ... CreatedLoadBalancer   Created load balancer
```

Here we see that we have an address of 104.196.248.204 now assigned to the 