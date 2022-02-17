High Availability pi-hole

I wanted a pi-hole that could survive nodes dying while keeping statistics and the same IP but could not find any tutorials on how to do that, so I made HApi-hole pronounced Happy Hole.

To set this up you will need 3 Raspberry pi 4's with 8G RAM and 2 high speed drives per pi. 4G RAM may be doable, but you are going to have issues with usage and need to know how to optimize your system to use as little RAM as possible. The OS drive has to be a fast drive that can handle a lot of writes, but the second drive on each pi could be a normal HDD.

I used:
* 3x https://www.amazon.com/gp/product/B089K47QDN/
* 3x https://www.amazon.com/gp/product/B07ZV1LLWK/
* 6x https://www.amazon.com/gp/product/B07S9CKV7X/
* 3x https://www.amazon.com/gp/product/B01N5IB20Q/
* 3x https://www.amazon.com/gp/product/B01N0TQPQB/

Image the 3 smaller SSD's with Ubuntu 20.10 64bit for raspberry pi and setup your pi's to boot from USB. You may need to use raspberry OS to update the firmware before USB boot works.

Connect the 3 OS drives into the top usb3.0 ports and the 3 blank ones onto the bottom usb3.0 ports on your pi's.

Make sure you have static IPs. Most these commands are ran on master01

Configure /etc/hosts on the three pi's so they can access eachother via hostname.
```bash
192.168.2.16 master01 master01.kube.local
192.168.2.17 master02 master02.kube.local
192.168.2.18 master03 master03.kube.local
```


Set the hostnames by running the following on the corrosponding pi's
```bash
sudo hostnamectl set-hostname master01.kube.local
sudo hostnamectl set-hostname master02.kube.local
sudo hostnamectl set-hostname master03.kube.local
```

make needed cgroup changes for k3s
```bash
sed -i '$ s/$/ cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
```

I'm not using wifi or bt on mine, so I disabled those to reduce power usage and set gpu memory usage to 16m
```bash
echo -e "dtoverlay=disable-wifi\ndtoverlay=disable-bt\ngpu_mem=16" >> /boot/firmware/config.txt
```

I overclocked since this is HA and if one pi dies, it is not a big deal, do this at your own risk
```bash
echo -e "over_voltage=6\narm_freq=2000\ngpu_freq=750" >> /boot/firmware/config.txt
```

Now we are going to update and do some installs.
```bash
sudo apt update
sudo apt upgrade -y
# if you are adding worker only nodes install only these packages.
sudo apt install libraspberrypi-bin rng-tools haveged net-tools sysstat lm-sensors smartmontools atop iftop -y
# master nodes get those as well as these packages
sudo apt install etcd docker.io ntp snapd ceph-common lvm2 cephadm build-essential golang git kubetail -y
```

Now we are going to get etcd working on the system.
```bash
mkdir /etc/etcd
touch /etc/etcd/etcd.conf
echo "ETCD_UNSUPPORTED_ARCH=arm64" > /etc/etcd/etcd.conf
sed -i '/\[Service]/a EnvironmentFile=/etc/etcd/etcd.conf' /lib/systemd/system/etcd.service
```

I am using ceph for the HA storage cluster but not using rook so I can keep it seperate and add nodes for only ceph with as little overhead as possible in the future.
```bash
mkdir -p /etc/ceph
# when adding the user, use a password you can remember, you will need it later
adduser cephuser
```

ceph needs no password sudo
```bash
usermod -aG sudo cephuser
touch /etc/sudoers.d/cephuser
echo "cephuser ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/cephuser
```

Now to restart. I like to do an update and upgrade just to be sure I did not forget so I do not have to restart again.
```bash
apt update; apt upgrade -y;reboot
```


Once the system is back up we are going to disable the etcd service so k3s can run its own.
```bash
systemctl disable etcd
systemctl stop etcd
```

Now to get ceph up and running. Replace --mon-ip with the IP you set for master01
```bash
cephadm bootstrap --skip-monitoring-stack --allow-fqdn-hostname --ssh-user cephuser --mon-ip 192.168.2.16
```

This may take a while, when it is done you will get a login url and admin user/pass. Log in and update the password.

Make sure a few settings are configured
```bash
ceph config set mgr mgr/cephadm/manage_etc_ceph_ceph_conf true
```

Copy the ceph SSH key to the other nodes. This is where you need to remember that password.
```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub cephuser@master02.kube.local
ssh-copy-id -f -i /etc/ceph/ceph.pub cephuser@master03.kube.local
```

Add your other nodes
```bash
ceph orch host add master02.kube.local
ceph orch host add master03.kube.local
```

Now to set the manager interface to run on all the nodes so you have access. You could pin this and I may see about running it separately in k3s to reduce resource usage later.
```bash
ceph orch apply mgr --placement="master01.kube.local master02.kube.local master03.kube.local"
```

Tell ceph to run the min of 3 monitors on your 3 master nodes. If you add master nodes in the future you can add them here. This is mainly so I can add OSD only nodes in the future and not have to worry about monitor services being started on it allowing me to use 4G RAM or lower nodes depending on the size of the disks.
```bash
ceph orch apply mon --placement="3 master01.kube.local master02.kube.local master03.kube.local"
```

Wait a while for all the hosts to be added. You can see this in the interface you logged into earlier.

Tell ceph to use any available free drives. You can specify the disk if you do not want to do it this way.
```bash
ceph orch apply osd --all-available-devices
```

If you do not see OSD's added in the interface and your cluster go to a healthy state, check if devices are available.
```bash
ceph orch device ls
```

If none of the storage drives are available for use, they may have a file system on them. Ceph requires them to no have any file systems. You can check with
```bash
lsblk
```

Make sure your OS is sda, and if your sdb has a blank fs you can clear it with
```bash
ceph orch device zap master01.kube.local /dev/sdb --force
```

Do that on all nodes. I find applying osd to all available again speeds up adding the drives.


Once ceph is running and drives added, we can install k3s. Run the following on the first master node.
```bash
curl -sfL http://get.k3s.io | sh -s - server --disable servicelb,traefik --cluster-init
```

Check if the node is in the Ready state.
```bash
sudo kubectl get nodes
```

Once the first master node is in the ready state, run this and run the output on additional master nodes waiting for each one to get to the ready state before doing the next one.
```bash
echo "curl -sfL http://get.k3s.io |K3S_TOKEN=`cat /var/lib/rancher/k3s/server/node-token` sh -s - server --disable servicelb,traefik --server https://`hostname -i`:6443"
```


If you have extra pi's you want to add as k3s worker nodes run the follwoing on the first master and the output on each worker node.
```bash
echo "curl -sfL http://get.k3s.io | K3S_URL=https://`hostname -i`:6443 K3S_TOKEN=`cat /var/lib/rancher/k3s/server/node-token` sh -"
```

I go the extra step of labeling worker nodes.
```bash
kubectl label node worker01.kube.local node-role.kubernetes.io/worker=''
```


Now I want some monitoring on the cluster, and I want my HA storage used so let's set up a ceph block device and k3s access to it.

Create the pool, initialize it, and add a user. You will get a "[client.kubernetes]" output from the last command with a key, copy this to a safe place as this is your user access key to that pool
```bash
ceph osd pool create kubernetes
rbd pool init kubernetes
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
```

Now we need to know the cluster id. The output of the following will include a fsid, this is your cluster id, copy it down to modify future commands.
```bash
ceph mon dump
```

Now we need a config map, replace the IPs with the ones you have set for your ceph mon nodes. This should be master01,02,03 if you did not change anything to this point. Also add the cluster id in the quotes.
```bash
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "ID_HERE",
        "monitors": [
          "192.168.2.16:6789",
          "192.168.2.17:6789",
          "192.168.2.18:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
```

Now apply that config
```bash
kubectl apply -f csi-config-map.yaml
```

Recent versions of ceph-csi also require an additional ConfigMap object to define Key Management Service (KMS) provider details.
```bash
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF
```

Now apply that config
```bash
$ kubectl apply -f csi-kms-config-map.yaml
```

Recent versions of ceph-csi also require yet another ConfigMap object to define Ceph configuration to add to ceph.conf file inside CSI containers
```bash
cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF
```

Now apply that config
```bash
kubectl apply -f ceph-config-map.yaml
```

Set up your access secret
```bash
cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes
  userKey: USER_KEY_HERE
EOF
```

apply it
```bash
kubectl apply -f csi-rbd-secret.yaml
```


```bash
cat <<EOF > kms-config.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {
      "vault-tokens-test": {
          "encryptionKMSType": "vaulttokens",
          "vaultAddress": "http://vault.default.svc.cluster.local:8200",
          "vaultBackendPath": "secret/",
          "vaultTLSServerName": "vault.default.svc.cluster.local",
          "vaultCAVerify": "false",
          "tenantConfigName": "ceph-csi-kms-config",
          "tenantTokenName": "ceph-csi-kms-token",
          "tenants": {
              "my-app": {
                  "vaultAddress": "https://vault.example.com",
                  "vaultCAVerify": "true"
              },
              "an-other-app": {
                  "tenantTokenName": "storage-encryption-token"
              }
          }
       }
    }
metadata:
  name: ceph-csi-encryption-kms-config
EOF
```


```bash
kubectl apply -f kms-config.yaml
```


We also need to set up a storage class, be sure to update the clusterID
```bash
cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: ID_HERE
   pool: kubernetes
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
mountOptions:
   - discard
EOF
```

```bash
kubectl apply -f csi-rbd-sc.yaml
```

Now we need our provisioner and plugin. You may not need rbac, I just add it anyway.
```bash
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
```


Make sure your storageclass is added
```bash
kubectl get storageclass
```

Watch and wait for all pods to start addressing any issues before proceeding.
```bash
watch "kubectl get pods --all-namespaces"
```


Once everything is good, I like to make the ceph rbd the default storage class
```bash
kubectl patch storageclass csi-rbd-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```


Now we are going to add metallb and get it setup.
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

You will need to make a config for it to work. I'm using the layer2 protocol. For the addresses, set a range that is not in your dhcp range. My dhcp is .100 and higher and I do static IPs below .50 so .50-.99 is safe for metallb to assign as it pleases.
```bash
cat <<EOF > metallb-config.yml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.2.50-192.168.2.99
EOF
```

```bash
kubectl apply -f metallb-config.yml
```


Make sure all your pods start up and now we are on to the actual monitoring part. I use:
https://github.com/carlosedp/cluster-monitoring

Read the settings in the vars.jsonnet. Here are a few settings I recommend
```bash

      name: 'armExporter',
      enabled: true,

      name: 'armExporter',
      enabled: true,

      name: 'traefikExporter',
      enabled: true,

  enablePersistence: {
    // Setting these to false, defaults to emptyDirs.
    prometheus: true,
    grafana: true,

    storageClass: 'csi-rbd-sc',
```

Since rbd is the default you do not technically need to set the storageclass. I also recommend increasing the size of the volumes if you do not know how to resize a pv and pvc as the defaults are a bit low and filled for me in 2 days. I do have 8 nodes though.


Once all the pods are running it is time to change grafana so you can access it via IP.
```bash
kubectl edit services -n monitoring grafana
```

Your default editor should be vim and you can use sed to update it from ClisterIP to LoadBalancer
```bash
:%s/type: ClusterIP/type: LoadBalancer/g
```

Now check what IP your service is running on.
```bash
kubectl get services -n monitoring grafana
```

In my case I get
```bash
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
grafana   LoadBalancer   10.43.227.221   192.168.2.50   3000:31370/TCP   9d
```
and can access grafana at http://192.168.2.50:3000


Now to move on to the pi-hole part of the HA pi-hole

I want to use cephfs for this for ReadWriteMany so we need to get that set up as well.


This will create the cephfs volume on ceph for you.
```bash
ceph fs volume create cephfs
```

Now k3s needs access
```bash
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-cephfsplugin-provisioner.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-cephfsplugin.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-nodeplugin-psp.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-nodeplugin-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-provisioner-psp.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-provisioner-rbac.yaml
```

We need a secret to access this. You should set up a user with access to cephfs, but I am just going to use admin for both admin and user. You can get the admin key from
```bash
cat /etc/ceph/ceph.client.admin.keyring
```


```bash
cat <<EOF > csi-cephfs-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default
stringData:
  userID: admin
  userKey: ADMIN_KEY
  adminID: admin
  adminKey: ADMIN_KEY
EOF
```

```bash
kubectl apply -f csi-cephfs-secret.yaml
```


Now we need a storage class. Be sure to update the cluster ID. It is the same as what you got before.
```bash
cat <<EOF > cephfs-csi-sc.yml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  clusterID: CLUSTER_ID
  fsName: cephfs
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - debug
EOF
```

```bash
kubectl apply -f cephfs-csi-sc.yml
```

Check your storageclass was added.
```bash
kubectl get storageclass
```

I like to make this the new default as it just makes things easier.
```bash
kubectl patch storageclass csi-rbd-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass csi-cephfs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Check storageclass again and make sure only the cephfs one is marked default.

```bash
kubectl get storageclass
```

For pi-hole I used this https://github.com/MoJo2600/pihole-kubernetes

I wanted my pi-hole to stay on the .99 IP, if you want a different one, update the config.

```bash
cat <<EOF > pihole.yaml
---
persistentVolumeClaim:
  enabled: true
  accessModes:
    - ReadWriteMany
serviceWeb:
  loadBalancerIP: 192.168.2.99
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc
  type: LoadBalancer

serviceDns:
  loadBalancerIP: 192.168.2.99
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc
  type: LoadBalancer

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
# If using in the real world, set up admin.existingSecret instead.
adminPassword: admin
DNS1: 1.1.1.1
DNS2: 8.8.8.8
EOF
```

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
snap install helm --classic
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
kubectl create namespace pihole
helm install --namespace pihole --values pihole.yaml pihole mojo2600/pihole
```

Installing this changes your resolv.conf to use pihole which makes it so k3s cannot resolve the dns to download it. You will want to remove the symlink /etc/resolv.conf and replace it with a normal file. Since I set my pihole to the .99 ip, I set this for my resolv.conf:

```
nameserver 192.168.2.99
nameserver 1.1.1.1
options edns0 trust-ad
search localdomain
```

With this the pods will try your pihole first, then fail over to 1.1.1.1

Now you should have pi-hole at
http://192.168.2.99/admin/
Some of the pods for this do not start and https access does not work. I have not had issues with http or the actual dns part of it.

You can test the HA aspect by finding which node the services is running on and shutting it down. Your ceph cluster should give you warnings but still work, and dns should work again in 8 minutes. There is a default 5 minute wait, then for me it takes 3 min to start on the new node.


I left ceph on defaults for rbd and cephfs in this case since with 3 nodes it will do a 3 copy replication meaning that each of your nodes has a copy of the data and with multi-master on k3s your pi-hole should survive 2 nodes dying.
