# Chapter 6: Labels and Annotations

Kubernetes was made to grow with you as your application scales in both size and complexity.
Labels and annotations are fundamental concepts in Kubernetes that let you work in sets of things that map to how you think about your application.
You can organize, mark, and cross-index all of your resources to represent the groups that make the most sense for your application.

Labels are key/value pairs that can be attached to Kubernetes objects such as Pods and ReplicaSets.
They can be arbitrary and are useful for attaching identifying information to Kubernetes objects.
Labels provide the foundation for grouping objects.

Annotations, on the other hand, provide a storage mechanism that resembles labels: key/value pairs designed to hold nonidentifying information that tools and libraries can leverage.
Unlike labels, annotations are not meant for querying, filtering, or otherwise differentiating Pods from each other.

## Labels

Labels provide identifying metadata for objects.
These are fundamental qualities of the object that will be used for grouping, viewing, and operating.
The motivations for labels grew out of Google's experience in running large and complex applications.
A couple of lessons emerged from this experience:

- Production abhors a singleton.
When deploying software, users often start with a single instance.
However, as the applicaiton matures, these singletons often multiply and become sets of objects.
With this in mind, Kubernetes uses labels to deal with sets of objects instead of single instances.
- Any hierarchy imposed by the system will fall short for many users.
In addition, user grouping and hierachies change over time.
For instance, a user may start out witht he idea that all apps are made up of many services.
However, over time, a service may be shared across multiple apps.
Kubernetes labels are flexible enough to adapt to these situtations and more.

Labels have simple syntax.
They are key/value pairs, where both the key and value are represented by strings.
Label keys can be broken down into two parts: an optional prefix and a name, seperated by a slash.
The prefix, if specified, must be a DNS subdomain with a 253-character limit.
The key name is required and have a maximum lenth of 63 characters.
Names must also start and end with an alphanumeric character and permit the use of dashes (-), underscores (_), and dots (.) between characters.

Label values are strings with a maximum length of 63 characters.
The contents of the label values follow the ssame rules as label keys.
Table 6-1 shows some valid label keys and values

| Key | value |
|--|--|
| acme.com/app-version | 1.0.0 |
| appVersion | 1.0.0 |
| app.version | 1.0.0 |
| kubernetes.io/cluster-service | true |

When domain names are used in labels and annotations, they are expected to be aligned to that particular entity in some way.
For example, a project might define a canonical set of labels used to identify the various stages of application deployment such as stagging, canary, and production.
Or a cloud provider might define provider-specific annotations that extend Kubernetes objects to activate features specific to their service.

### Applying Labels

Here we create a few deployments (a way to create an array of Pods) with some interesting labels.
We'll take two apps (called `alpaca` and `bandicoot`) and have two environments and two versions for each.

First, create the `alpaca-prod` deployment and set the `ver`, `app`, and `env` labels:

```bash
kubectl run alpaca-prod \
--image=gcr.io/kuar-demo/kuard-amd64:blue \
--replicas=2 \
--labels="ver=1,app=alpaca,env=prod"
```

Next, cretae the `alpaca-test` deployment and set the `ver`, `app`, and `env` labels with the appropriate values:

```bash
kubectl run alpaca-test \
--image=gcr.io/kuar-demo/kuard-amd64:green \
--replicas=1 \
--labels="ver=2,app=alpaca,env=test"
```

Finally create two deployments for bandicoot.
Here we name the environment `prod` and `staging`:

```bash
kubectl run bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --replicas=2 \
  --labels="ver=2,app=bandicoot,env=prod"
$ kubectl run bandicoot-staging \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --replicas=1 \
  --labels="ver=2,app=bandicoot,env=staging"
```

At this point you should have four deployments--alpaca-prod, alpaca-test, bandicoot-prod, and bandicoot-staging:

```bash
$ kubectl get deployments --show-labels

NAME                ... LABELS
alpaca-prod         ... app=alpaca,env=prod,ver=1
alpaca-test         ... app=alpaca,env=test,ver=2
bandicoot-prod      ... app=bandicoot,env=prod,ver=2
bandicoot-staging   ... app=bandicoot,env=staging,ver=2
```

### Modifying Labels

You can also apply or update labels on objects after you create them:

`kubectl label deployments alpaca-test "canary-true`

**WARNING**

There is a caveat here.
In this example, the kubectl label command will only change the lavel on the deployment itself; it won't affect any objects that the deployment creates, such as ReplicaSets and Pods.
To change those, you'll need to change the template embedded in the deployment

**EOW**

You can also use the `-L` option to `kubectl get` to show a label value as a column:

```bash