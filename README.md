
# Deploying a Flask application on a local Kubernetes cluster (Minikube)

Step-by-step instructions for a demo on how to deploy a Flask application on a local Kubernetes cluster (minikube).

## Pre-requisites

Install a container runtime and a kubernetes flavour. In this example, docker cli, docker compose, minikube and colima are installed on a M1 Macbook.

```bash
brew install minikube
brew install docker docker-compose
```

Run this before procedding with colima installation
```bash
mkdir -p ~/.docker/cli-plugins
ln -sfn /opt/homebrew/opt/docker-compose/bin/docker-compose ~/.docker/cli-plugins/docker-compose
```
At this point we'll have the docker command but it won't be any daemon to actually run containers.
```bash
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

To install the daemon to run the containers we'll use colima.
```bash
brew install colima
```

Start the container daemon using colima start
```bash
$ colima start

INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0000] preparing network ...                         context=vm
INFO[0000] creating and starting ...                     context=vm
INFO[0070] provisioning ...                              context=docker
INFO[0070] starting ...                                  context=docker
INFO[0076] done
```

Using colima list we check whether is running.
```bash
$ colima list

PROFILE    STATUS     ARCH       CPUS    MEMORY    DISK     RUNTIME    ADDRESS
default    Running    aarch64    2       2GiB      60GiB    docker
```
Once it is running we can check with docker ps that it can connect to the daemon.
```bash
$ docker ps

CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Configure minikube with the docker driver.
```bash
minikube config set driver docker
```

Start minikube
```bash
$ minikube  start
😄  minikube v1.29.0 on Darwin 12.6.2 (arm64)
✨  Using the docker driver based on user configuration
📌  Using Docker Desktop driver with root privileges
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.26.1 preload ...
    > preloaded-images-k8s-v18-v1...:  330.51 MiB / 330.51 MiB  100.00% 5.09 Mi
    > gcr.io/k8s-minikube/kicbase...:  368.75 MiB / 368.75 MiB  100.00% 3.07 Mi
🔥  Creating docker container (CPUs=2, Memory=1976MB) ...
🐳  Preparing Kubernetes v1.26.1 on Docker 20.10.23 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```

Open k8s dashboard
```bash
$ minikube dashboard

🔌  Enabling dashboard ...
    ▪ Using image docker.io/kubernetesui/dashboard:v2.7.0
    ▪ Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
💡  Some dashboard features require the metrics-server addon. To enable all features please run:

	minikube addons enable metrics-server


🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:52023/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

Enable ingress
```bash
$ minikube addons enable ingress

💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
💡  After the addon is enabled, please run "minikube tunnel" and your ingress resources would be available at "127.0.0.1"
    ▪ Using image registry.k8s.io/ingress-nginx/controller:v1.5.1
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```

## Container build and test locally

Build the container
```bash
docker build -t flask-app-test .
```

Run locally
```bash
docker run --name test-flask -p 5000:5000 flask-app-test
```
Note: Cant run on Mac M1 because of 5000 port being used by AirPlay - [Access to localhost was denied You don't have authorisation to view this page. HTTP ERROR 403](https://stackoverflow.com/questions/70913242/access-to-localhost-was-denied-you-dont-have-authorisation-to-view-this-page-h)

Open browser and point to http://localhost:5000 . It should display an Instance ID.

Load the image into minikube since it's available locally.

```bash
minikube image load flask-app-test
```

## Deployment

Apply the manifest file.

```bash
kubectl apply -f kubernetes/flask_deployment.yaml
```
Check the deployment

```bash
$ kubectl get deploy

NAME        READY   UP-TO-DATE   AVAILABLE   AGE
flask-app   5/5     5            5           6s
```
Check the pods
```bash
$ kubectl get pod

NAME                         READY   STATUS    RESTARTS   AGE
flask-app-6dc69d44f9-b5bx9   1/1     Running   0          21s
flask-app-6dc69d44f9-bqjch   1/1     Running   0          21s
flask-app-6dc69d44f9-fwhsw   1/1     Running   0          21s
flask-app-6dc69d44f9-hvjlj   1/1     Running   0          21s
flask-app-6dc69d44f9-vgzld   1/1     Running   0          21s
```
Check the logs from one of the pods
```bash
$ kubectl logs flask-app-6dc69d44f9-b5bx9

[2023-02-15 18:47:55 +0000] [1] [INFO] Starting gunicorn 20.1.0
[2023-02-15 18:47:55 +0000] [1] [INFO] Listening at: http://0.0.0.0:5001 (1)
[2023-02-15 18:47:55 +0000] [1] [INFO] Using worker: sync
[2023-02-15 18:47:55 +0000] [7] [INFO] Booting worker with pid: 7
```
Scale deployment to 10 pods
```bash
$ kubectl scale deployment flask-app --replicas=10

deployment.apps/flask-app scaled
```
Check deployment again
```bash
$ kubectl get deploy

NAME        READY   UP-TO-DATE   AVAILABLE   AGE
flask-app   10/10   10           10          5m43s
```

## Service

```bash
kubectl apply -f kubernetes/flask_service.yaml

service/flask-app-service created
```

Check the services
```bash
$ kubectl get svc

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
flask-app-service   ClusterIP   10.111.178.32   <none>        5001/TCP   42s
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP    35m
```
SSH into the node to test the cluster IP
```bash
$ minikube ssh

docker@minikube:~$ curl 10.111.178.32
```

## Ingress
```bash
$ kubectl apply -f kubernetes/flask_ingress.yaml

ingress.networking.k8s.io/flask-app-ingress created
```

Get ingress
```bash
$ kubectl get ing

NAME                CLASS    HOSTS   ADDRESS        PORTS   AGE
flask-app-ingress   <none>   *       192.168.49.2   80      11m
```

Point browser to http://192.168.49.2 and refresh the page to see the Instance IDs changing based on the pod that the request is directed to.


## Cleanup

Delete the deployment

```bash
kubectl delete deploy flask-app
```

Delete the service
```bash
kubectl delete svc flask-app-service
```

Delete the ingress
```bash
kubectl delete ing flask-app-ingress
```
