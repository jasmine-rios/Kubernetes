# Chapter 13: ConfigMaps and Secrets

It's good practice to make container images as resuble as possible.
The same image should be able to be used for development, staging, and production.
It's even better if the same image is general-purpose enough to be used across applications and services.
Testing and versioning are more risky and complicated if images need to be re-created for each new environment.
How then do we specialize the use of that image at runtime?

This is where ConfigMaps and Secrets come into play.
ConfigMpas ar used to provide configuration information for workloads.
This can be either fine-grained information like a string or a composite value in the form of a file.
Secrets are similar to ConfigMaps but focus on making sensitive information available to the workload.
They can be used for things like credentials or TLS certificates.

## ConfigMaps

One way to think of a ConfigMap is a kubernetes object that defines a small filesystem.
Another way is a set of variables that can be used when defining the environment or command line for your containers.
The key thing to note is that the ConfigMap is combined with the Pod right before it is run.
This means that the container image and the Pod definition can be reused by many workloads just by changing the ConfigMap that is used.

### Creating ConfigMaps

Let's jump right in and create a ConfigMap.
Like many objects in Kubernetes, you can create these in an immediate, imperative way, or you can create them from a manifest on disk.
We'll start with the imperative method.

First, suppose we have a file on disk (called myconfig.txt) that we want to make available to the Pod in question:

```yaml
# This is a sample config file that I might use to configure an application
parameter1 = value1
parameter2 = value2
```

Next, let's create a ConfigMap with that file.
We'll also add a couple of simple key/value pairs here.
These are referred to as literal values on the command line:

```bash
$ kubectl create configmap my-config \
  --from-file=my-config.txt \
  --from-literal=extra-param=extra-value \
  --from-literal=another-param=another-value
```

The equivalent YAML for the ConfigMap object we just created is as follows:

```yaml
$ kubectl get configmaps my-config -o yaml

apiVersion: v1
data:
  another-param: another-value
  extra-param: extra-value
  my-config.txt: |
    # This is a sample config file that I might use to configure an application
    parameter1 = value1
    parameter2 = value2
kind: ConfigMap
metadata:
  creationTimestamp: ...
  name: my-config
  namespace: default
  resourceVersion: "13556"
  selfLink: /api/v1/namespaces/default/configmaps/my-config
  uid: 3641c553-f7de-11e6-98c9-06135271a273
```

As you can see, the ConfigMap is just some key/value pairs stored in an object.
The interesting part is when you try to use a ConfigMap.

### Using a ConfigMap

There are three main ways to use a ConfigMap:

    Filesystem
        You can mount a ConfigMap into a Pod.
        A file is created for each entry based on the key name.
        The contents of that file are set to the value.
    
    Environment variable
        A ConfigMap can be used to dynamically set the value of an environment variable.
    
    Command-line argument
        Kubernetes support dynamically creating the command line for a container based on ConfigMap values.

Let's create a manifest for `kuard` that pulls all of these together:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-config
spec:
  containers:
    - name: test-container
      image: gcr.io/kuar-demo/kuard-amd64:blue
      imagePullPolicy: Always
      command:
        - "/kuard"
        - "$(EXTRA_PARAM)"
      env:
        # An example of an environment variable used inside the container
        - name: ANOTHER_PARAM
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: another-param
        # An example of an environment variable passed to the command to start
        # the container (above).
        - name: EXTRA_PARAM
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: extra-param
      volumeMounts:
        # Mounting the ConfigMap as a set of files
        - name: config-volume
          mountPath: /config
  volumes:
    - name: config-volume
      configMap:
        name: my-config
  restartPolicy: Never
```

For the filesystem method, we create a new volume inside the Pod and give it the name `Config-volume`.
We then define this volume to be a ConfigMap volume and point at the ConfigMap to mount.
We have to specify where this gets mounted into the `kuard` container with a `volumeMount`.
In this case, we are mounting it at `/config`.

Environment varaibles are specified with a special `valueFrom` member.
This references the ConfigMap and the data key to use within that ConfigMap.
Command-line arguments build on environment variables.
Kubernetes will perform the correct substitution with a special `$(<env-var-name>)` syntax.

Run this Pod, and let's port-forward to examine how the app sees the world:

```bash
$ kubectl apply -f kuard-config.yaml
$ kubectl port-forward kuard-config 8080
```

Now point your browser to http://localhost:8080.
We can look at how we've injected configuration values into the program in all three ways.
Click the "Server Env" tab on the left.
This will show the command line that the app was launched with along with its environment.

Here we can see that we've added two environment variables (`ANOTHER_PARAM` and `EXTRA_PARAM`) whose values are set via the ConfigMap.
We've also added an agrument to the command line of `kuard` based on the `EXTRA_PARAM` value.

Next, click the "File system browser" tab.
This let's you explore the filesystem as the application sees it.
You should see an entry called `/config`.
This is a volume created based on our ConfigMap.
If you navigate into that, you'll see that a file has been created for each entry of the ConfigMap.
You'll also see some hidden files (prepended with..) that are used to do a clean swap of new values when the ConfigMap is updated.