# gke-pwx-example

### relative links
- machine types
  - https://cloud.google.com/compute/docs/machine-types
- zones
  - https://cloud.google.com/compute/docs/regions-zones/#available  
- regions
  - https://cloud.google.com/compute/docs/regions-zones/#available
- clusters
  - https://cloud.google.com/kubernetes-engine/docs/how-to/managing-clusters
- kubectl
  - https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl
- etcd TLS
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/cluster_tls.md
- etcd backup operator
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/walkthrough/backup-operator.md
- etcd restore operator
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/walkthrough/restore-operator.md

## Setup
```
▶gcloud config set project <project>

▶gcloud config set compute/region <region> (needed for regional/multi-az)

▶gcloud config set compute/zone <zone>

Update
▶ gcloud components update
```

## Create a cluster (regional/multizone)
```
gcloud container clusters create test-cluster-ryan \
--region us-east1 \
--node-locations us-east1-b,us-east1-c,us-east1-d \
--disk-type=pd-ssd \
--disk-size=50GB \
--labels=portworx=gke \
--machine-type=n1-standard-8 \
--num-nodes=3 \
--image-type ubuntu \
--enable-autoscaling --max-nodes=6 --min-nodes=3
```

## Create a cluster (single zone)
```
gcloud container clusters create test-cluster-ryan \
--zone us-east1-d \
--disk-type=pd-ssd \
--disk-size=50GB \
--labels=portworx=gke \
--machine-type=n1-standard-4 \
--num-nodes=3 \
--image-type ubuntu \
--enable-autoscaling --max-nodes=6 --min-nodes=3
```

## Manage your cluster

Describe cluster
`gcloud container clusters describe test-cluster-ryan`

Set default cluster
`gcloud config set container/cluster test-cluster-ryan`

Get endpoint
```
gcloud container clusters describe test-cluster-ryan | grep endpoint
endpoint: 35.231.17.210
```

Configure kubectl
`gcloud container clusters get-credentials test-cluster-ryan`

Get nodes
```
kubectl get no
NAME                                               STATUS    ROLES     AGE       VERSION
gke-test-cluster-ryan-default-pool-ac8eed24-c38h   Ready     <none>    16m       v1.9.7-gke.6
gke-test-cluster-ryan-default-pool-ac8eed24-clh1   Ready     <none>    15m       v1.9.7-gke.6
gke-test-cluster-ryan-default-pool-ac8eed24-ftf7   Ready     <none>    16m       v1.9.7-gke.6
```

## Deploy Portworx

### Enable Compute API
`gcloud services enable compute.googleapis.com`

Create a ClusterRoleBinding for your user using the following commands:

### Get current google identity
``` 
gcloud info | grep Account
Account: [myname@example.org]
```

### Grant cluster-admin to your current identity
```
kubectl create clusterrolebinding myname-cluster-admin-binding \
  --clusterrole=cluster-admin --user=myname@example.org
Clusterrolebinding "myname-cluster-admin-binding" created
```

## Deploy Etcd

Create Secretes for TLS
```
kubectl create secret -n kube-system generic etcd-peer-tls --from-file=etcd/certs/peer-ca.crt --from-file=etcd/certs/peer.crt --from-file=etcd/certs/peer.key

kubectl create secret -n kube-system  generic etcd-server-tls --from-file=etcd/certs/server-ca.crt --from-file=etcd/certs/server.crt --from-file=etcd/certs/server.key

kubectl create secret -n kube-system  generic etcd-client-tls --from-file=etcd/certs/etcd-client-ca.crt --from-file=etcd/certs/etcd-client.crt --from-file=etcd/certs/etcd-client.key
```

Deploy Operator
```
kubectl apply -f etcd/rbac/cluster-role.yaml
kubectl apply -f etcd/rbac/cluster-role-binding.yaml
kubectl apply -f etcd/operator/etcd-operator.yaml
```

Wait until its ready
```
kubectl get pods -o wide -n kube-system -l name=etcd-operator
NAME                             READY     STATUS    RESTARTS   AGE       IP          NODE                                               NOMINATED NODE
etcd-operator-768dc99865-p2l7p   1/1       Running   0          22s       10.12.1.7   gke-test-cluster-ryan-default-pool-ac8eed24-c38h   <none>
```

Deploy Etcd
`kubectl apply -f etcd/etcd-cluster.yaml`

Monitor if its ready
```
kubectl get pods -o wide -n kube-system -l app=etcd
▶ kubectl get pods -o wide -n kube-system -l app=etcd
NAME                               READY     STATUS    RESTARTS   AGE       IP          NODE                                               NOMINATED NODE
portworx-etcd-cluster-hc7w6cfxtv   1/1       Running   0          49s       10.12.0.6   gke-test-cluster-ryan-default-pool-ac8eed24-ftf7   <none>
portworx-etcd-cluster-qfnqg6jj7q   1/1       Running   0          33s       10.12.2.6   gke-test-cluster-ryan-default-pool-ac8eed24-clh1   <none>
portworx-etcd-cluster-trckvhxp9w   1/1       Running   0          1m        10.12.1.8   gke-test-cluster-ryan-default-pool-ac8eed24-c38h   <none>
```

Gather the Etcd endpoint that portworx needs
```
kubectl describe svc portworx-etcd-cluster-client -n kube-system
Name:              portworx-etcd-cluster-client
Namespace:         kube-system
Labels:            app=etcd
                   etcd_cluster=portworx-etcd-cluster
Annotations:       service.alpha.kubernetes.io/tolerate-unready-endpoints=true
Selector:          app=etcd,etcd_cluster=portworx-etcd-cluster
Type:              ClusterIP
IP:                10.15.240.61
Port:              client  2379/TCP
TargetPort:        2379/TCP
Endpoints:         10.12.0.6:2379,10.12.1.8:2379,10.12.2.6:2379
Session Affinity:  None
Events:            <none>
```

Create PX Spec
```
VER=$(kubectl version --short | awk -Fv '/Server Version: /{print $3}')
export ETCD="etcd://10.15.240.61:2379"
curl -L -o px-spec.yaml "https://install.portworx.com/1.4/?c=mycluster&k=${ETCD}&kbver=$VER&stork=true&s=type=pd-ssd,size=200&st=k8s"
```

## TODO
- show get etcd ep
- px install/spec
- pxctl output
- ha failover example
- postgres example
- 3d snap example
- cloudsnap
- restore
- scale