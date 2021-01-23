# Deploy K8S on baremetal servers using RKE and MetalLB


MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.

From https://metallb.universe.tf/

```text
Why?
Kubernetes does not offer an implementation of 
network load-balancers (Services of type LoadBalancer) for bare metal clusters. 

The implementations of Network LB that Kubernetes does ship with are all glue code that 
calls out to various IaaS platforms (GCP, AWS, Azure…). 
If you’re not running on a supported IaaS platform (GCP, AWS, Azure…), 
LoadBalancers will remain in the “pending” state indefinitely when created.

Bare metal cluster operators are left with two lesser tools to bring user traffic into their clusters, 
“NodePort” and “externalIPs” services. 
Both of these options have significant downsides for production use, 
which makes bare metal clusters second class citizens in the Kubernetes ecosystem.

MetalLB aims to redress this imbalance by offering a Network LB implementation that 
integrates with standard network equipment, 
so that external services on bare metal clusters also “just work” as much as possible.
```

From your local machine, ensure that you can ssh to all baremetal servers using a private key

My baremetal server are ve1301, ve1302, ve1303, ve1304.

```bash
[marc@marcrhel82 ~]$ ssh -i /home/marc/id_rsa marc@ve1301
Enter passphrase for key '/home/marc/id_rsa':
[marc@ve1301 ~]$
```

On each baremetal server, ensure that your user is part of the docker group

```bash
su marc
sudo groupadd docker
sudo usermod -aG docker marc
newgrp docker
```

On each baremetal server, verify that you can run docker commands without sudo.

```bash
[marc@ve1301 ~]$ cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)
[marc@ve1301 ~]$ docker run hello-world

Hello from Docker!
```


On your local machine, get the latest RKE e.g.

```bash
wget https://github.com/rancher/rke/releases/download/v1.2.4/rke_linux-amd64
sudo mv rke_linux-amd64 /usr/bin/rke
sudo chmod +x /usr/bin/rke
```

From your local machine, deploy K8S on your baremetal servers using RKE

```bash
[marc@marcrhel82 ~]$  rke up --config cluster.yaml --ssh-agent-auth --ignore-docker-version
```

My cluster config used above is at https://github.com/marcredhat/rke/blob/main/cluster.yaml


Check that K8S nodes are ready

```bash
[marc@marcrhel82 ~]$ KUBECONFIG=kube_config_cluster.yaml kubectl get nodes
NAME                        STATUS     ROLES               AGE   VERSION
ve1301 Ready      controlplane,etcd   14h   v1.19.6
ve1302 Ready      worker              14h   v1.19.6
ve1303 Ready      worker              14h   v1.19.6
ve1304 Ready      worker              14h   v1.19.6
```

Add the generated kubeconfig to your .bashrc

```bash
export KUBECONFIG=/home/marc/kube_config_cluster.yaml
```

```bash
source .bashrc
```

## Deploy MetalLB

```bash
kubectl create namespace metallb-system
kubectl config set-context --namespace  metallb-system --current
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
```

On first install only

```bash
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

On your baremetal servers, identify the IP network that MetalLB will be able to allocate addresses from 

```
[root@ve1301 ~]# ip a | grep docker0
15: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    inet 172.19.0.1/16 brd 172.19.255.255 scope global docker0


My MetalLB pool configuration can be found at https://github.com/marcredhat/rke/blob/main/addresspoolcm.yaml

The ConfigMap above tells MetalLB to allocate IP address from 172.19.27.230-172.19.27.250

```bash    
kubectl apply -f https://raw.githubusercontent.com/marcredhat/rke/main/addresspoolcm.yaml
```


Create a Deployment

```bash
oc apply -f https://raw.githubusercontent.com/marcredhat/kind/main/deploy.yaml
```

Expose the Deployment as type LoadBalancer

```bash
oc expose deploy nginx-web  --type=LoadBalancer
```

Check that MetalLB allocated an external IP from the range we specified in the ConfigMap

```bash
[marc@marcrhel82 ~]$ oc get svc -o wide
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE   SELECTOR
nginx-web   LoadBalancer   10.xx.xx.xx   172.19.27.230   8080:30486/TCP   12h   app=nginx-web
```

From any of your baremetal servers, access the service via the external IP

```bash
[root@ve1302 ~]# curl  172.19.27.230:8080
Hello, world!
Version: 1.0.0
Hostname: nginx-web-7675865c58-b5lfw
```



