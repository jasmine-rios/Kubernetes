# Kubernetes

## Start minikube cluster

Use this command to start minikube
`minikube start --driver docker`

You can check the status of minikube
`minikube status`

## Commands to Apply

```bash
kubectl get pod
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f webapp.yaml
```

## Interacting with Kubernetes

```bash
kubectl get all
```

Webapp-service is listed as NodePort type meaning we can access it from a web browser.

We don't see configmap or secret.
We can access it using:

```bash
kubectl get configmap
kubectl get secret
kubectl get pod
```

If you need help use
`kubectl --help`

You can also get help on command using
`kubectl get --help`

Get detailed information with "describe" command
`kubectl describe service webapp-service`

Get detailed info on pods
`kubectl describe pod <pod-name>`

View logs of container
`kubectl logs <pod-name>`

Stream logs using -f option
`kubectl logs <pod-name> -f`

## Access application from Browser

Use the command to get the IP and port of the service
`kubectl get svc`

Get IP address of mini-kube
`minikube ip`

You can use this to get info on minikube
`kubectl get node`

You can get wider output
`kubectl get node - o wide`

Get the minikube ip and Web-service node port and go to a browser and enter:
<minikube ip>:30100
