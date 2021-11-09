![Logo of the project](https://www.egov-nsdl.co.in/img/logo.png)

# NSDL Edastakhat
> Esign Everywhere

A brief description of your project, what it is used for and how does life get
awesome when someone starts to use it.

## Installing / Getting started

### Install Docker
>Note: Login docker using specific account
```shell
docker login
```
### Install Microk8s Latest Version Kubernetes Single Node Cluster
>Note: This will install lastest version of cluster. Any change in version may affect other resources too
```shell
sudo snap install microk8s --classic
```

### Install Microk8s Specific Version Kubernetes Single Node Cluster
>Note: In Development Enviornment we have installed 1.22 version
```shell
sudo snap install microk8s --classic --channel=1.22.2/stable
```

Above command will install docker and kubernetes cluster.

### Initial Configuration

Require some initial configuration after installation of cluster.
Create `docker.json inside` `/etc/docker/docker.json`
```shell
{"insecure-registries" : ["127.0.0.1:32000", "172.19.74.42:32000", "172.30.151.90:32000"]}
```
Edit Registry file inside microk8s cluster.
Edit `/var/snap/microk8s/current/args/containerd-template.toml` and add the below lines under `[plugins] -> [plugins."io.containerd.grpc.v1.cri".registry]`.
This will inform docker to register these endpoind as a insecure docker registry.

```shell
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."172.19.74.42:32000"]
          endpoint = ["http://172.19.74.42:32000"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."127.0.0.1:32000"]
          endpoint = ["http://127.0.0.1:32000"]
```
to make these changes effective execute below commands this will restart microk8s cluster.
```
microk8s stop
microk8s start
```
## Setup K8s Single Node Microk8s Cluster

Note: For errorfree deployment, please follow orddering of the steps mentioned.

```shell
alias kubectl='microk8s.kubectl'
microk8s enable rbac
microk8s enable dns:172.19.65.167
microk8s enable registry:size=40Gi
microk8s enable traefik
```

And state what happens step-by-step.

## Create namespaces

```shell
kubectl create namespace db
kubectl create namespace dt
```
db namespace will be used for Postgres db and Activemq
dt namspace will be used for Zipkin,Grafana and Prometheus

## Setup traefik Edge Router

Below commands will install traefik role based access, custom resource defination, ingress routes for edastakhat.
Middlleware for handling traefik routes.

```shell
kubectl apply -f /ed-k8s/deploy/dev/traefik/traefik-rbac.yml
kubectl apply -f /ed-k8s/deploy/dev/traefik/traefik-crd.yml
kubectl apply -f /ed-k8s/deploy/dev/traefik/ed-ingress.yml
kubectl apply -f /ed-k8s/deploy/dev/traefik/ed-middleware.yml
```
After installation of traefik, edit traefik daemonsets and add below config.
```shell
      --providers.kubernetesingress=false
      --providers.kubernetesingress.ingressendpoint.ip=127.0.0.1
      --log=true
      --log.level=INFO
      --accesslog=true
      --accesslog.filepath=/dev/stdout
      --accesslog.format=json
      --entrypoints.web.address=:80
      --entrypoints.websecure.address=:8443
      --entrypoints.postgres.address=:5432
      --providers.kubernetescrd
      --api.dashboard=true
      --api
      --api.insecure
```

## Install traefik dashboard
```
kubectl apply -f /ed-k8s/deploy/dev/traefik/traefik-dashboard.yml
```

What's traefik dashboard?
* What's the main functionality
* You can also do another thing
* If you get really randy, you can even do this

## Install Edastakhat Resources and Postgres 13.4 db

The Edastakhat Project.
### Install secrets, global config maps, volumes for Postgres db

```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-global-cm.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-secrets.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-postgres-secrets.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-postgres-pv-pvc.yml
```
### Postgres 13.4 database Deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-postgres-deploy.yml
```
list resource in db namespace
```shell
kubectl get all -n db
```
logs in db pod
```shell
kubectl -n db logs pod/postgres-84c49bfc85-nvxkn
```
output:
```shell
PostgreSQL Database directory appears to contain a database; Skipping initialization

2021-10-30 06:00:30.246 UTC [1] LOG:  starting PostgreSQL 13.4 on x86_64-pc-linux-musl, compiled by gcc (Alpine 10.3.1_git20210424) 10.3.1 20210424, 64-bit
2021-10-30 06:00:30.246 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2021-10-30 06:00:30.246 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2021-10-30 06:00:30.580 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2021-10-30 06:00:31.003 UTC [23] LOG:  database system was shut down at 2021-10-29 14:02:12 UTC
2021-10-30 06:00:31.270 UTC [1] LOG:  database system is ready to accept connections
```
## Activemq messaging queue deployment

#### Install persistance volume and volume claim
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-activemq-pv-pvc.yml
```
#### Deploy Activemq
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-activemq-deploy.yml
```

## Redis deployment

Deploy Redis
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-redis.yml
```
check redis logs:
```
kubectl -n db logs pod/redis-0
```
output:
```
1:C 30 Oct 2021 06:33:02.114 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 30 Oct 2021 06:33:02.114 # Redis version=6.2.3, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 30 Oct 2021 06:33:02.114 # Configuration loaded
1:M 30 Oct 2021 06:33:02.116 * monotonic clock: POSIX clock_gettime
1:M 30 Oct 2021 06:33:02.117 * Running mode=standalone, port=6379.
1:M 30 Oct 2021 06:33:02.118 # Server initialized
1:M 30 Oct 2021 06:33:02.118 * Ready to accept connections
```
## Edastakhat Eureka Deployment
### Eureka Discovery Server
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-registry.yml
```
Eureka 2 instance should be up, wait for a moment and check browser Eureka cluster running or not.
Check Eureka instance are ready
- Eureka: http://eureka.dev.edastakhat.nsdl.com

## Edastakhat Config Server Deployment
Config server target are located in mount path of deployment.

### Deploy PV, PVC and deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-config-pv-pvc.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-config.yml
```
Check config server logs before proceed ahead.
Check all config loaded from github.

## Edastakhat API Gateway Deployment
### Deployment Spring Cloud Gateway
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-agw.yml
```
- Check API gateway: http://agw.dev.edastakhat.nsdl.com

## Edastakhat Authorization Server Deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-auth.yml
```
## Edastakhat Invoice Service Deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-inv-pv-pvc.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-invoice.yml
```
## Edastakhat Service Deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-pv-pvc.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-edastakhat.yml
```
## Edastakhat Captcha Service Deployment
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-captcha.yml
```
## Edastakhat Stamping Service
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-stamp-pv-pvc.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-stamp.yml
```
## Edastakhat SMS Service
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-sms.yml
```
## Edastakhat Email Service
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-email.yml
```
## Edastakhat OTP Service
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-otp-pv-pvc.yml
kubectl apply -f /ed-k8s/deploy/dev/ed-otp.yml
```
## Edastakhat UI
```shell
kubectl apply -f /ed-k8s/deploy/dev/ed-ui.yml
```
- Edastakhat: http://agw.dev.edastakhat.nsdl.com/ed/

## Kubernetes Dashboard Configuration
Read below blogs:
- Ref1: http://blog.zachinachshon.com/k8s-dashboard/
- Ref2: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

Our cluster is enabled with RBAC.
We need to create a user for dashboard access.
Create ServiceAccount, ClusterRole and ClusterRoleBinding
```shell
microk8s kubectl apply -f  /k8s/sa-k8s-admin-user.yml
```
```shell
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 --address 0.0.0.0
microk8s kubectl -n kube-system describe sa dashboard-admin
microk8s kubectl -n kube-system describe secrets dashboard-admin-token-kvqw2
```
### Kubernetes Dashboard SSL config
Create tls certificate using openssl.
- Ref1: https://github.com/kubernetes/dashboard/blob/master/docs/user/certificate-management.md
- Ref2: https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#recommended-setup

list Kubernetes dashboard services
```shell
kubectl -n kube-system create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs
microk8s kubectl edit deployment.apps/kubernetes-dashboard  -n kube-system
```
Add args in deployment and restart kubernetes dashboard deployment.

```shell
- --tls-cert-file=/tls.crt
- --tls-key-file=/tls.key
```
## Traefik websecure SSL port change
Edit traefik daemonset and change websecure port from 8443 to 443.
Edit daemonset and change port.
```shell
microk8s kubectl -n traefik edit daemonset.apps/traefik-ingress-controller
```

## Contributing

When you publish something open source, one of the greatest motivations is that
anyone can just jump in and start contributing to your project.

These paragraphs are meant to welcome those kind souls to feel that they are
needed. You should state something like:

"If you'd like to contribute, please fork the repository and use a feature
branch. Pull requests are warmly welcome."

If there's anything else the developer needs to know (e.g. the code style
guide), you should link it here. If there's a lot of things to take into
consideration, it is common to separate this section to its own file called
`CONTRIBUTING.md` (or similar). If so, you should say that it exists here.

## Install Microk8s on centos7
### Install snap package manager on centos7

- https://docs.fedoraproject.org/en-US/epel/
- https://snapcraft.io/docs/installing-snap-on-red-hat
- https://computingforgeeks.com/install-microk8s-kubernetes-cluster-on-centos/
- https://phoenixnap.com/kb/configure-centos-network-settings

### After install of microk8s
>Note: Access right must be granted.
```shell
sudo usermod -aG microk8s $USER
sudo chown -f -R $USER ~/.kube
```
### Change Centos hostname
>Note: hostname should be as per Open Cloud specification otherwise kubernetes will normalize hostname and this may give unpredictable result.
```shell
hostnamectl status
hostnamectl set-hostname ed-kmaster
nproc
free -m
```
## Edastakhat cluster links

- API Gateway: http://agw.dev.edastakhat.nsdl.com
- Activemq Console: http://activemq.dev.edastakhat.nsdl.com
- Postgres DB: http://postgres.dev.edastakhat.nsdl.com
- Zipking Tracing dashboard: http://zipkin.dev.edastakhat.nsdl.com
- Grafana Dashboard: http://grafana.dev.edastakhat.nsdl.com
- Eureka Dashboard: http://eureka.dev.edastakhat.nsdl.com
- Traefik Dashboard: http://traefik.dev.edastakhat.nsdl.com
- Redis: http://redis.dev.edastakhat.nsdl.com
- Kubernetes Dashboard: http://k8s.dev.edastakhat.nsdl.com

## Links and References

Even though this information can be found inside the project on machine-readable
format like in a .json file, it's good to include a summary of most useful
links to humans using your project. You can include links like:

- Project homepage: https://your.github.com/awesome-project/
- Repository: https://github.com/your/awesome-project/
- Issue tracker: https://github.com/your/awesome-project/issues
  - In case of sensitive bugs like security vulnerabilities, please contact
    my@email.com directly instead of using issue tracker. We value your effort
    to improve the security and privacy of this project!
- Related projects:
  - Your other project: https://github.com/your/other-project/
  - Someone else's project: https://github.com/someones/awesome-project/


## Licensing

NSDL eGovernance Ltd
