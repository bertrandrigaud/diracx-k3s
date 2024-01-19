# diracx-k3s

Deploy diracx on a k3s cluster remotely


## Resources

kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/  
  
helm: https://helm.sh/docs/intro/install/  
  
k3sup: https://github.com/alexellis/k3sup  
k3s: https://docs.k3s.io/  
  
loghorn: https://longhorn.io/  
  
diracx: https://github.com/DIRACGrid/diracx  
diracx-charts: https://github.com/DIRACGrid/diracx-charts  


## Requirements

- Accessible cluster machines via ssh

- kubectl (client for managing kubernetes cluster)

- helm (tool for managing kubernetes deployments via charts)

Install kubectl (on laptop)
---------------------------

```
# kubectl
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl

# kubectl checksum file
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# validate binary
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

Install helm (on laptop)
---------------------------

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Enable completion (optional but useful)
---------------------------------------

```
# kubectl
source <(kubectl completion bash)

# helm
source <(helm completion bash)
```


## Deploy K3S remotely (using k3sup)

Install k3sup (on laptop)
-------------------------

```
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

Assuming your cluster is composed of 2 machines (main server and agent server)

```
# install k3s on main server

export SERVER_IP=xxx.xxx.xxx.xxx
export USER=root

k3sup install --ip $SERVER_IP --user $USER


# join agent server

export AGENT_IP=xxx.xxx.xxx.xxx

k3sup join --ip $AGENT_IP --server-ip $SERVER_IP --user $USER
```

Storage configuration (Longhorn)
--------------------------------

open-iscsi is required. Depending on your cluster OS, follow this install procedure:
https://longhorn.io/docs/1.5.3/deploy/install/#installing-open-iscsi  

example for RHEL, Centos:
```
yum --setopt=tsflags=noscripts install iscsi-initiator-utils
echo "InitiatorName=$(/sbin/iscsi-iname)" > /etc/iscsi/initiatorname.iscsi
systemctl enable iscsid
systemctl start iscsid
```

Then deploy longhorn in your cluster:  
```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.1/deploy/longhorn.yaml
```

Note: Check `storageClassName: longhorn` is set in `diracx-charts/diracx/templates/cs-store-volume.yml`


Test your cluster
-----------------

```
export KUBECONFIG=`pwd`/kubeconfig
kubectl config use-context default
kubectl get node

# k3s comes with pods already deployed 
kubectl get pods -A
```

## Deploy Kubernetes Dashboard (optional but useful)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

kubectl apply -f ./manifest/dashboard/cluster-role.yaml
kubectl apply -f ./manifest/dashboard/secret.yaml 
kubectl apply -f ./manifest/dashboard/service-account.yaml
```

```
# generate token
kubectl -n kubernetes-dashboard create token admin-user
```

```
# launch web server
kubectl proxy
```

In a web browser, go to : http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/  
Note: use token created just above for login  
  
Choose `Token` as login method, paste the token just generated  

## Get Traefik Dashboard

Traefik comes out of the box with k3s. In order to access Traefik Dashboard from your laptop:   

```
kubectl --namespace kube-system port-forward deployments/traefik 9000:9000 &
```

In a web browser, go to : http://localhost:9000/dashboard/  


## Deploy diracx

Clone diracx repositories
-------------------------
```
git clone https://github.com/DIRACGrid/diracx.git
git clone https://github.com/DIRACGrid/diracx-charts.git
```


Deploy via provided helm charts
-------------------------------
```
helm install --kubeconfig ./kubeconfig diracx ./diracx-charts/diracx/ -f ./diracx-charts/demo/values.tpl.test.yaml --debug
```

