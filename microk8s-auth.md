## STEP I check cluster version and dir creations
```
microk8s kubectl version
microk8s kubectl get nodes
sudo mkdir kubeusers & cd kubeusers
openssl genrsa -out niks.key 2048
openssl req -new -key niks.key -out niks.csr -subj '/CN=niks/O=devlopment'
```

## STEP II go to micok8s location
```
/var/snap/microk8s/current/certs
cp /var/snap/microk8s/current/certs/ca.{crt,key} .
openssl x509 -req -in niks.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out niks.crt -days 365
```

## STEP III view cluster info
```
microk8s kubectl config view
```

## STEP IV
```
microk8s kubectl --kubeconfig niks.kubeconfig config set-cluster microk8s-cluster --server https://172.19.74.42:16443 --certificate-authority=ca.crt
microk8s kubectl --kubeconfig niks.kubeconfig config set-credentials niks --client-certificate niks.crt --client-key niks.key
microk8s kubectl --kubeconfig niks.kubeconfig config set-context niks-kubernetes --cluster microk8s-cluster --namespace default --user niks
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
```
kubectl create role --help
kubectl create role niks-default --verb=get,list,create,delete,watch,* --resource=* --namespace=default
kubectl get role niks-default -o yaml -n default
```
## STEP VIII
```
kubectl create rolebinding --help
kubectl create rolebinding niks-default-rolebinding --role=niks-default --user=niks --namespace=default
kubectl get rolebinding niks-default-rolebinding -o yaml -n default
```
(if problem with while accessing resources with this role, edit the created role and modifiy it)
kubectl -n default edit role niks-default

## STEP VI
```
microk8s kubectl --kubeconfig niks.kubeconfig get pods
```
## Convert Keys to base64
```
cat niks.key | base64 -w0
cat niks.crt | base64 -w0
cat ca.crt | base64 -w0
```
