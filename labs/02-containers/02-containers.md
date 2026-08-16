# 02-containers.md

## Create your own container

Build the application yourself with
`docker build -t kubernetes-demo:v1 .`

Then test it locally with 
`docker run -p 8080:8080 kubernetes-demo:v1`

^This command gave me an error so I checked `docker images`.
I didn't see anything so I went in and created the `dockerfile` in Visual Studios Code in the folder `app`.

I created this file structure 

```
Kubernetes/
├── README.md
├── .gitignore
│
├── app/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── kubernetes/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
│
├── kind/
│   └── cluster-config.yaml
│
├── helm/
│   └── kubernetes-demo/
│
├── argocd/
│   └── application.yaml
│
├── monitoring/
│   ├── prometheus/
│   └── grafana/
│
├── scripts/
│
├── labs/
│   ├── 01-cluster-fundamentals/
│   ├── 02-containers/
│   ├── 03-pods/
│   ├── 04-deployments/
│   └── 05-networking/
│
└── .github/
    └── workflows/
        └── ci.yaml
```

I added this to app/Dockerfile

```
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["python", "app.py"]
```

I put this for app/requirements.txt

```
Flask==3.0.3
```

I put this for app/app.py
```
from flask import Flask
import socket

app = Flask(__name__)

@app.route("/")
def home():
    return {
        "application": "Kubernetes GitOps Demo",
        "version": "v1",
        "hostname": socket.gethostname(),
        "status": "Healthy"
    }

@app.route("/health")
def health():
    return {
        "status": "healthy"
    }, 200

@app.route("/ready")
def ready():
    return {
        "status": "ready"
    }, 200

if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=8080
    )
```

I went to `cd app` in terminal and ran `ls` and seen:
- Dockerfile
- app.py
- requirements.txt

Then I ean `docker build-t kubernetes-demo:v1`

I still got this error:

```bash
docker build -t kuber
netes-demo:v1
ERROR: "docker buildx build" requires exactly 1 argument.
See 'docker buildx build --help'.

Usage:  docker buildx build [OPTIONS] PATH | URL | -

Start a build
```

So I did `docker build -t kubernetes-demo:v1 .` 
and it worked.

Then I ran
`docker run -p 8080:8080 kubernetes-demo:v1`

It should say something in the output like
```bash
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8080
 * Running on http://172.17.0.2:8080
```

