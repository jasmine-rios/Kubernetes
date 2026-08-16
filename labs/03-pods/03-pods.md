# 03-pods.md

We will deploy a pod with declarative language via a yaml file.

Create pods.yaml and put in this content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-demo
spec:
  containers:
    - name: kubernetes-demo
      image: kubernetes-demo:v1
```

Then in terminal do a
`kubectl apply -f pods.yaml`

Inspect it with `kubectl get pods` and see there is a pod called `kubernetes-demo`.

Use `kubectl describe pod kubernetes-demo`

You can see the IP of the Pod `10.244.0.5` and that ip address belongs to `kubernetes-demo` inside the Kubernetes Pod network.

This is different than the node IP mentioned in 02-containers.md

Run `kubectl logs kuberentes-demo`
and I got an error
```
$ kubectl logs kuber netes-demo 

Error from server (BadRequest): container "kubernetes-demo" in pod "kubernetes-demo" is waiting to start: trying and failing to pull image
```

Because my Pod was still in ImagePullBackOff
so Kubernetes was failing before it reaches the point where app.py can run and produce logs

To fix this I did `docker images | grep kubernetes-demo`
and it showed up.

I then had to load that local image into the kind cluster
`kind load docker-image kubernetes-demo:v1 --name gitops-lab`