## STEP I check cluster version and dir creations
```
microk8s kubectl version
microk8s kubectl get nodes
sudo mkdir kubeusers & cd kubeusers
openssl genrsa -out niksa.key 2048
openssl req -new -key niksa.key -out niksa.csr -subj '/CN=niksa/O=devlopment'
```

## STEP II go to micok8s location
>Note: Copy cluster keys (certificates) to sign new CSR
```
/var/snap/microk8s/current/certs
cp /var/snap/microk8s/current/certs/ca.{crt,key} .
openssl x509 -req -in niksa.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out niksa.crt -days 365
```

## STEP III view cluster info
```
microk8s kubectl config view
```

## STEP IV
```
microk8s kubectl --kubeconfig niks.kubeconfig config set-cluster microk8s-cluster --server https://172.19.74.42:16443 --certificate-authority=ca.crt
microk8s kubectl --kubeconfig niks.kubeconfig config set-credentials niksa --client-certificate niksa.crt --client-key niksa.key
microk8s kubectl --kubeconfig niks.kubeconfig config set-context niks-kubernetes --cluster microk8s-cluster --namespace default --user niksa
```

## STEP V edit config file niks.kubeconfig
```
vi niks.kubeconfig
change this line 'server: https://127.0.0.1:16443'
add/update this line 'current-context: "niks-kubernetes"'
```

## STEP VI
```
microk8s kubectl --kubeconfig niks.kubeconfig get pods
```

## STEP VII
### Create Role
>Note: For Role based access RBAC must be enabled in cluster
```
kubectl create role --help
kubectl create role niks-default --verb=get,list,create,delete,watch,* --resource=* --namespace=default
kubectl get role niks-default -o yaml -n default
```

## STEP VIII
### Create Rolebinding
>Note: For Role based access RBAC must be enabled in cluster
```shell
kubectl create rolebinding --help
kubectl create rolebinding niks-default-rolebinding --role=niks-default --user=niks --namespace=default
kubectl get rolebinding niks-default-rolebinding -o yaml -n default
```
(if problem with while accessing resources with this role, edit the created role and modifiy it)
```
kubectl -n default edit role niks-default
```

## STEP VI list pods with kubecconfig file
```shell
microk8s kubectl --kubeconfig niks.kubeconfig get pods
```
## STEP X
### Create Service Account
```shell
microk8s kubectl create serviceaccount dashboard-user
```
### Create ClusteRrrole and ClusterRoleBinding
>Note: For ClusterRole based access RBAC must be enabled in cluster
```shell
microk8s kubectl create clusterrole niksa-pod-reader --verb=get,list,watch --resource=*
microk8s kubectl create clusterrolebinding niksa-pod-reader --clusterrole=niksa-pod-reader --user=niksa --serviceaccount=default:dashboard-user
```
### Get token from service account to login dashboard with kubeconfig file
```shell
microk8s kubectl  describe sa dashboard-user
microk8s kubectl  describe secrets dashboard-user-token-4dhcr
```
## Convert Keys to base64
```
cat niksa.key | base64 -w0
cat niksa.crt | base64 -w0
cat ca.crt | base64 -w0
```
