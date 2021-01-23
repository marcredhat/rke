# Deploy K8S on baremetal servers using RKE and MetalLB


From your local machine, ensure that you can ssh to all baremetal servers using a private key

My baremetal server are ve1301, ve1302, ve1303, ve1304.

```bash
[marc@marcrhel82 ~]$ ssh -i /home/marc/id_rsa marc@ve1301
Enter passphrase for key '/home/marc/id_rsa':
[marc@ve1301 ~]$
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
