# Prerequisties

## OS specific

already running nfs service

```
$ showmount -e 192.168.88.57
Export list for 192.168.88.57:
/mnt/md0/terramaster *
```

nfs common utils 

```
dpkg -l | grep nfs
ii  libnfsidmap2:amd64                     0.25-5.1                                        amd64        NFS idmapping library
ii  nfs-common                             1:1.3.4-2.1ubuntu5.5                            amd64        NFS support files common to client and server
```

this is already installed as part of ansible rol `k8squickbase`

# k8s storge class

need to get helm - `get_helm.sh` at root of this repo

install the helm chart for nfs subdir external provisioner :

```
echo "installing helm chart..."
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
echo "installing nfs-subdir-external-provisioner.."
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.88.57 \
    --set nfs.path=/mnt/md0/terramaster
```    

# manifests

##

physical volume

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs-client
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /mnt/md0/terramaster
    server: 192.168.88.57
```

physical volume claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: nfs-client
```

create a pod that uses the claim

```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: myclaim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

# file structure 

```
$ sudo mount -t nfs 192.168.88.57:/mnt/md0/terramaster /mnt
....

$ tree /mnt/
/mnt/
├── archived-default-myclaim-pvc-c73c9c93-f0ba-43e6-a4a2-34908faf2869
│   └── test_folder
│       └── message_of_importance.txt
├── default-myclaim-pvc-47f7efc6-16ca-455a-9809-71efc7cd32f0
│   └── me.txt
├── default-myclaim-pvc-ccab07c0-5329-41bc-8582-965294e263ef
└── #recycle
```

if a pv is deleted, it gets archived

as new pvc's are created, a new 'default-.....' mount point is created

this default policay ( see helm variables ) makes new directories for each 'volume' and 'claim' - suggesting we can have as many pv and pvcs in a single nfs volume as we care for - minding to keep track of the name for each pv

need to do some more work with claims - to see if we can use fractions of the same pv by size - simplest is to uese the whole volume by claim and volume with the same size

