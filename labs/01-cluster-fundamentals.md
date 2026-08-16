# 01 cluster fundamentals

## Dependencies

If running a MacOS
Run `brew install <software-name>`

And replace <software-name> with the following:
- Docker
- kubectl
- kind
- Git
- Github account
- Helm
- Argo
## Create your cluster

Run your first cluster
`kind create cluster --name gitops-lab`

**IMPORTANT**
Had to do this in order to run command:
`open -a docker`

The Docker desktop will come up

verify using `docker ps` and there should be nothing there but just a table will no values.

You can also run `docker info`.

After all those commands I was able to run:
`kind create cluster --name gitops-lab`

Click allow if there is a popup.

You should now see the cluster by entering 
`kubectl cluster-info --context kind-gitops-lab`

It should say that CoreDNS is running at https://127.0.0.1:64315/api/v1/namespace/kube-system/services/kube-dns:dns/proxy

The Ip address is where your local machine can reach the Kuberenetes API server for this `kind` cluster.

`kind` normally binds to API server at `127.0.01` and you can explicitly configure a port, it can select an avilable port automatically

If you go to `127.0.0.1:64315` in your web browser, nothing will show.

Now run `kubectl cluster-info` and it will show the same thing as well.

Check if you have nodes by `kubectl get nodes` and you see that you have a control plane node named `gitops-lab-control-plane`

Inspect it further with `kubectl get nodes -o wide`

You will see the name, status (should be `ready` by now), the role it has `control-plane` and the version of Kubernetes it is running `v.1.36.1` and it's OS-image `Debian GNU/Linux 13`, and It's internal IP `172.18.0.2`.
It will aldo show its OS Image `6.3.13-linuxkit (arm64)` and it's container-runtime `containerd://2.3.1`

Type in `kubectl get namespaces` to see that there was multiple name spaces created:

- default
- kube-node-lease
- kube-public
- kube-system
- local-path-storage

Run `kubectl get pods -A` to see the pods created:

For kube-system you see pods for kub-system:
- coredns (2)
- etcd 
- kindnet 
- kube proxy


kube-apiserver-gitops-lab-control-plane:
- api server
- controller manager
- kube-scheduler

local-path-storage:
- path provisioner


Then run `kubectl api-resources` to see all the things you can create by reaching out to the API server (e.g., ConfigMaps, namespaces, nodes, pods. secrets, services, policies, deployments, volumes, role, and rolebindings)

## Create your own container

Build the application yourself with
`docker build -t kubernetes-demo:v1 .`

Then test it locally with 
`docker run -p 8080:8080 kubernetes-demo:v1`

^This command gave me an error so I checked `docker images`.
I didn't see anything so I went back to the folder that contains my `dockerfile`